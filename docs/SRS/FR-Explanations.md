# FR Explanations

---

## FR-PLT-01

**FR-PLT-01** is the first and most foundational requirement — it creates a new tenant (school) in the system. When a Platform Admin calls `POST /api/admin/tenants`, they provide:

| Field | Explanation |
|-------|-------------|
| **name** | School name (e.g., "Springfield School") |
| **subdomain** | Unique slug used for the tenant's login URL (`{subdomain}.sirius-skool.com`). Must be globally unique. |
| **modules** | Array of module keys that pre-authorizes this tenant to use specific features. School Admin can toggle these on/off later via Module Management, but cannot enable modules outside this list. [Read more below](#fr-plt-01.1). |
| **sms_quota** | The number of SMS credits pre-allocated to the tenant. This acts as a prepaid wallet for sending text message notifications (e.g., absence alerts, result publication). Each SMS sent deducts 1 credit. When the balance reaches 0, SMS sending is blocked but email continues to work. Platform Admin can top up later via FR-PLT-09. [Read more below](#fr-plt-01.2). |
| **starting_sequence** | The starting value for student registration numbers (e.g., `1` → first student gets `26000001`). [Read more below](#fr-plt-01.3). |
| **school_admin** | Initial admin credentials — `full_name`, `email`, `password`. This user is auto-created in the `users` table with role `SCHOOL_ADMIN`. |

**What happens atomically (BR-PLT-01, single transaction):**
1. `tenants` row created (with `is_active: true`, `current_student_sequence: <starting_sequence>`)
2. `users` row created for the School Admin
3. `tenant_settings` row created (empty/defaults)
4. `tenant_modules` rows seeded for each assigned module

**Validation:**
- Subdomain uniqueness → `409 SUBDOMAIN_TAKEN`
- Invalid fields → `422 VALIDATION_ERROR`

<a id="fr-plt-01.1"></a>
#### FR-PLT-01.1 Module Pre-Authorization — Detailed Breakdown

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

This is the gate to everything — no module or feature works until a tenant exists.

---

<a id="fr-plt-01.2"></a>
#### FR-PLT-01.2 SMS Credit System — Detailed Breakdown

The SMS credit system is a prepaid balance model that controls how many text messages a tenant can send:

| Concept | Detail |
|---------|--------|
| **Where stored** | `tenants.sms_balance` column (integer, default = `sms_quota` at creation) |
| **Initial balance** | Set by `sms_quota` during tenant creation. Stored in `sms_balance`. |
| **Consumption rate** | 1 credit per SMS sent. Applies to absent alerts (FR-ATT-06), result notifications (FR-NOT-02), and ad-hoc sends (FR-NOT-04). |
| **When deducted** | Deducted atomically after the SMS provider confirms successful delivery (FR-NOT-05). |
| **When blocked** | Balance = 0 → `402 SMS_QUOTA_EXCEEDED`. Email is NOT affected. |
| **Top-up** | Platform Admin adds credits via PATCH `/api/admin/tenants/{id}/sms-balance` (FR-PLT-09). |
| **Negative protection** | Balance can never go below 0. Deduction attempts that would underflow are rejected. |
| **Audit trail** | Every adjustment and deduction is logged in `audit_logs` (FR-AUD-05). |

**Example lifecycle:**
1. Tenant created with `sms_quota: 500` → `sms_balance = 500`
2. Daily attendance: 5 absent students → 5 SMS sent → `sms_balance = 495`
3. Results published: 30 SMS sent to guardians → `sms_balance = 465`
4. Over weeks, balance gradually depletes to `sms_balance = 0`
5. Next SMS attempt → `402 SMS_QUOTA_EXCEEDED` — admin sees warning on dashboard (FR-DSH-01)
6. Platform Admin adds 200 credits → `sms_balance = 200` — SMS resumes

<a id="fr-plt-01.3"></a>
#### FR-PLT-01.3 Registration Number Sequence — Detailed Breakdown

`starting_sequence` is the initial value for the tenant's student registration counter. Each time a student is admitted, the counter is atomically read, used, and incremented by 1.

**Registration number format:** `YY` (2-digit admission year) + `NNNNNN` (6-digit zero-padded sequence)

- Example: `26000001` = admitted in 20**26**, sequence **000001**
- The `YY` prefix changes with the admission year, but **the sequence counter never resets** — it increments perpetually per tenant

**How the sequence counter works:**

| Concept | Detail |
|---------|--------|
| **Stored in** | `tenants.current_student_sequence` column |
| **Initial value** | Set to `starting_sequence` at tenant creation |
| **Increment** | +1 after each successful student creation |
| **Atomicity** | Read + increment uses `SELECT ... FOR UPDATE` to prevent race conditions (BR-STU-02) |
| **Scope** | Per-tenant, continuous across all academic years |
| **Permanence** | Registration numbers never change, even if a student leaves or graduates (BR-STU-03) |

**Example lifecycle (starting_sequence = 1):**

| # | Event | Year | Sequence read | Registration No |
|---|-------|------|---------------|-----------------|
| 1 | First admission | 2026 | 1 | `26000001` |
| 2 | Second admission | 2026 | 2 | `26000002` |
| ... | ... | ... | ... | ... |
| 150 | 150th admission | 2026 | 150 | `26000150` |
| 151 | First admission next year | 2027 | 151 | `27000151` |
| 152 | Another admission | 2027 | 152 | `27000152` |

Notice: the `YY` prefix changed from `26` to `27`, but the sequence counter continued from 151.

**Why make it configurable?** `starting_sequence` allows schools migrating from a legacy system to avoid conflicts with existing student IDs. For example, if a school already has 500 students in their old system and is migrating to Sirius-Skool, they can set `starting_sequence: 501` so new registration numbers start at `26000501` instead of `26000001`.

## FR-PLT-02

**FR-PLT-02** ensures that each tenant's subdomain is globally unique, preventing URL collisions and tenant impersonation. When a Platform Admin submits a tenant creation request, the system checks the `tenants` table for an existing row with the same subdomain before proceeding.

**Validation:**
- Subdomain already exists → `409 SUBDOMAIN_TAKEN`

---

## FR-PLT-03

**FR-PLT-03** automatically creates the School Admin user during tenant creation. The `users` table gets a new row with the provided name, email, hashed password, `role = SCHOOL_ADMIN`, and `tenant_id` pointing to the newly created tenant. This happens in the same database transaction (BR-PLT-01) — if user creation fails, the entire tenant creation rolls back.

---

## FR-PLT-04

**FR-PLT-04** creates a `tenant_settings` record for every new tenant with default values (empty logo, address, phone, email). This guarantees that BR-SET-02 ("Every tenant has exactly one settings record") is satisfied from day one. School Admin can update these settings later via the Settings module (FR-SET-02).

---

## FR-PLT-05

**FR-PLT-05** seeds `tenant_modules` rows for all modules assigned by the Platform Admin during tenant creation. Each assigned module gets a record with `is_enabled = true`. This determines the pool of modules the School Admin can later toggle on/off — they cannot enable modules not assigned here (BR-MM-02).

---

## FR-PLT-06

**FR-PLT-06** allows the Platform Admin to activate or deactivate a tenant via `PATCH /api/admin/tenants/{id}`. The request body includes `{ "is_active": false }` to deactivate. Deactivation immediately blocks all user access (FR-PLT-07). Reactivation restores login capability, but previous sessions are not restored. [Read more below](#fr-plt-06.1).

<a id="fr-plt-06.1"></a>
#### FR-PLT-06.1 Activate/Deactivate — Detailed Breakdown

**FR-PLT-06** allows the Platform Admin to toggle a tenant's active status via `PATCH /api/admin/tenants/{id}`. This is how the Platform Admin controls which schools can access the system.

**Request:**
```json
{ "is_active": false }
```

**What happens on deactivation:**
1. `tenants.is_active` set to `false`
2. All subsequent login attempts for that tenant return `403 TENANT_INACTIVE` (FR-PLT-07)
3. Existing JWT tokens become effectively unusable — the auth middleware checks tenant status on each request
4. Active sessions are not proactively revoked, but each request validates tenant activity

**What happens on reactivation:**
1. `tenants.is_active` set to `true`
2. Users can log in again normally
3. Previous sessions are not restored — users must re-authenticate

**Why PATCH instead of PUT?**

PATCH is used because the update is **partial** — you're only changing `is_active`, not replacing the entire tenant resource.

| Method | Behavior | Example |
|--------|----------|---------|
| **PATCH** | Partial update — only send the fields that change | `PATCH /api/admin/tenants/{id}` with `{ "is_active": false }` |
| **PUT** | Full replacement — must send every field | Would require `{ "name": "...", "subdomain": "...", "is_active": false, ... }` |

This follows REST conventions: PATCH for partial modification, PUT for full replacement.

**Validation:**
- Tenant not found → `404`
- Only Platform Admin can call this endpoint

**Edge cases:**

| Scenario | Behavior |
|----------|----------|
| Deactivate active tenant | All users blocked immediately. Middleware checks on each API request detect the inactive status and return 403. |
| Reactivate deactivated tenant | Login resumes. Users who had valid JWTs when the tenant was deactivated will find them expired by the time of reactivation (15-min access token window). |
| Deactivate already-deactivated tenant | No-op — `is_active` stays `false`, response returns `{ "is_active": false }`. |
| Attendance session in progress during deactivation | The session save fails with 403. Data integrity is maintained — no partial writes. |

---

## FR-PLT-07

**FR-PLT-07** enforces tenant deactivation at the authentication layer. When `tenants.is_active = false`, all login attempts return `403 TENANT_INACTIVE`. Existing JWT tokens are checked against tenant status on each request via middleware, making deactivation effectively immediate.

---

## FR-PLT-08

**FR-PLT-08** provides the Platform Admin a list of all tenants with their name, subdomain, active status, SMS balance, and creation date via `GET /api/admin/tenants`. This is the main tenant management overview.

---

## FR-PLT-09

**FR-PLT-09** allows the Platform Admin to adjust a tenant's SMS balance via `PATCH /api/admin/tenants/{id}/sms-balance`. The adjustment can be positive (add credits) or negative (deduct). Negative adjustments are validated to prevent the balance from going below zero. [Read more below](#fr-plt-09.1).

<a id="fr-plt-09.1"></a>
#### FR-PLT-09.1 SMS Balance Adjustment — Detailed Breakdown

**FR-PLT-09** allows the Platform Admin to add or deduct SMS credits for a tenant. This controls how many text message notifications the school can send.

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

**Why PATCH?** Same reasoning as FR-PLT-06 — partial update. You're only changing `sms_balance`, not the entire tenant record. The endpoint path `/sms-balance` makes the specific resource being modified explicit.

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

---

## FR-PLT-10

**FR-PLT-10** provides a cross-tenant notification log view for the Platform Admin via `GET /api/admin/notifications`. This enables monitoring of SMS/Email delivery status across all tenants for troubleshooting and auditing.

---

## FR-PLT-11

**FR-PLT-11** allows the Platform Admin to reset any tenant user's password (School Admin or Manager) without requiring the current password. This is the only way a tenant user can get a password reset — there is no self-service forgot-password flow.

**How it works:**

1. Platform Admin calls `PATCH /api/admin/users/{id}/reset-password` with `{ "new_password": "NewPass123" }`
2. System updates the user's `password_hash` in the `users` table
3. System increments `token_version` to invalidate all existing sessions (BR-AUTH-08)
4. User must log in again with the new password

**Validation:**
- User not found → `404 USER_NOT_FOUND`
- Only Platform Admin → `403 FORBIDDEN`

---

## FR-AUTH-01

**FR-AUTH-01** authenticates Platform Admin credentials against the `platform_admins` table at the dedicated admin login page (`admin.sirius-skool.com`). When the user submits `POST /api/admin/auth/login`, the system verifies the email and bcrypt-hashed password, then issues a JWT access token and refresh token on success.

**Validation:**
- Invalid email or password → `401 INVALID_CREDENTIALS`
- Rate limit exceeded → `429 RATE_LIMIT_EXCEEDED`

---

## FR-AUTH-02

**FR-AUTH-02** authenticates School Admin and Manager credentials against the `users` table, scoped to the tenant identified by the subdomain in the HTTP Host header. The tenant is resolved from the URL (e.g., `springfield.sirius-skool.com` → tenant with subdomain `springfield`), and credentials are checked only within that tenant's user records. [Read more below](#fr-auth-02.1).

<a id="fr-auth-02.1"></a>
#### FR-AUTH-02.1 Tenant-Scoped Authentication — Detailed Breakdown

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

---

## FR-AUTH-03

**FR-AUTH-03** extracts the tenant identifier from the HTTP Host header before authentication occurs. For example, a request to `springfield.sirius-skool.com` extracts `springfield` as the subdomain, which is used to look up the `tenants` record. The `X-Tenant-Slug` header provides a fallback for local development environments. [Read more below](#fr-auth-03.1).

<a id="fr-auth-03.1"></a>
#### FR-AUTH-03.1 Tenant Resolution — Detailed Breakdown

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
4. If tenant found but inactive → handled by FR-AUTH-13 (`403 TENANT_INACTIVE`)

---

## FR-AUTH-04

**FR-AUTH-04** issues a JWT access token (15-minute expiry) and a UUID refresh token (7-day expiry) on successful authentication. The access token is sent on every API request for authorization. The refresh token enables session extension without requiring the user to re-enter credentials. [Read more below](#fr-auth-04.1).

<a id="fr-auth-04.1"></a>
#### FR-AUTH-04.1 Token Issuance — Detailed Breakdown

**FR-AUTH-04** issues two tokens on successful login: a short-lived **access token** for API authorization and a longer-lived **refresh token** for session extension.

**Token pair:**

| Token | Type | Lifetime | Storage | Purpose |
|-------|------|----------|---------|---------|
| **Access token** | JWT | 15 minutes | Client (in-memory/header) | Sent on every API request in `Authorization: Bearer <token>` header. Contains user ID, tenant ID, role, and token version. |
| **Refresh token** | UUID v4 | 7 days | Client + Server (DB or signed JWT) | Used only at `POST /api/auth/refresh` to get a new access token when the current one expires. |

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

**Open question (Q-AUTH-02):** Refresh token storage is undecided — either a `refresh_tokens` DB table (revocable, queryable) or signed JWTs without DB lookup (faster, no storage). This decision affects the revocation strategy and reuse detection implementation (FR-AUTH-07).

---

## FR-AUTH-05

**FR-AUTH-05** redirects users to their role-specific dashboard after successful login. Platform Admin → `/admin/dashboard`. School Admin and Manager → their tenant dashboard (`/dashboard`), with the Manager view scoped to their assigned permissions. [Read more below](#fr-auth-05.1).

<a id="fr-auth-05.1"></a>
#### FR-AUTH-05.1 Role-Based Redirection — Detailed Breakdown

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

---

## FR-AUTH-06

**FR-AUTH-06** revokes the refresh token on logout via `POST /api/auth/logout`. The provided refresh token is invalidated in storage, making it unusable for future refresh attempts. The user must re-authenticate to obtain new tokens. [Read more below](#fr-auth-06.1).

<a id="fr-auth-06.1"></a>
#### FR-AUTH-06.1 Logout & Token Revocation — Detailed Breakdown

**FR-AUTH-06** invalidates the refresh token when a user explicitly logs out, ensuring the session cannot be resumed.

**How it works:**

1. User clicks "Logout" → frontend sends `POST /api/auth/logout` with the current refresh token
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

---

## FR-AUTH-07

**FR-AUTH-07** issues new tokens via `POST /api/auth/refresh` using refresh token rotation with reuse detection. [Read more below](#fr-auth-07.1).

<a id="fr-auth-07.1"></a>
#### FR-AUTH-07.1 Token Refresh, Rotation & Reuse Detection — Detailed Breakdown

**FR-AUTH-07** extends the session by issuing new tokens when the access token expires. It uses **rotation** (old token revoked, new one issued) and **reuse detection** (stolen token triggers full revocation).

**Normal flow:**
1. Access token expires (~15 min) → frontend calls `POST /api/auth/refresh` with the current refresh token
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

---

## FR-AUTH-08

**FR-AUTH-08** allows School Admin and Platform Admin to change their own password via `POST /api/auth/change-password`. The user provides their current password and a new password. The system verifies the current password before updating. Manager role is blocked from using this endpoint.

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

---

## FR-AUTH-09

**FR-AUTH-09** returns the current authenticated user's profile, tenant information, and effective permissions via `GET /api/auth/me`. Platform Admin sees platform-level data. School Admin sees their tenant info plus all enabled module permissions. Manager sees their tenant info plus their assigned action-level permissions.

---

## FR-AUTH-10

**FR-AUTH-10** blocks login for deactivated users. When `users.is_active = false`, the login endpoint returns `403 ACCOUNT_INACTIVE` regardless of whether the password is correct. This applies to both School Admin and Manager accounts.

---

## FR-AUTH-11

**FR-AUTH-11** blocks login for deactivated tenants. When `tenants.is_active = false`, all login attempts for that tenant return `403 TENANT_INACTIVE`. This prevents access to all users (School Admin + Managers) in the deactivated tenant.

---

## FR-AUTH-12

**FR-AUTH-12** implements rate limiting on the login endpoint — 5 failed attempts per minute per IP address. When exceeded, the endpoint returns `429 RATE_LIMIT_EXCEEDED` and additional attempts are blocked until the window resets. This mitigates brute-force attacks.

---

## FR-ACA-01

**FR-ACA-01** allows School Admin to create academic years (e.g., "2026-2027") with a start date and end date via `POST /api/academic-years`. Academic years are the top-level entity that scopes enrollments, attendance, exams, and other time-bound data.

**What happens:**
- A new `academic_years` row is created with the provided name and dates
- If `is_current` is set to `true`, the system ensures no other academic year is current (FR-ACA-02)

**Validation:**
- Duplicate name → `409 DUPLICATE_NAME`
- Overlapping dates → `409 OVERLAPPING_DATES`
- Another year already current → `400 CURRENT_YEAR_EXISTS`

---

## FR-ACA-02

**FR-ACA-02** enforces that a tenant has exactly one active (current) academic year at any time. When a School Admin sets `is_current = true` on an academic year, all other years for that tenant are automatically set to `is_current = false`. This ensures that dashboards, reports, and data queries have an unambiguous current context.

---

## FR-ACA-03

**FR-ACA-03** prevents academic year date ranges from overlapping. The system validates start_date and end_date against all existing academic years for the tenant. Overlapping years would create ambiguity for enrollment and attendance recording.

---

## FR-ACA-04

**FR-ACA-04** allows School Admin to create classes (e.g., "Class 6", "Class 10") with a short code, display name, and sort order via `POST /api/classes`. Classes are permanent entities reused across academic years — they are soft-disabled rather than deleted (BR-ACA-04).

---

## FR-ACA-05

**FR-ACA-05** allows School Admin to create sections within a class (e.g., "Section A", "Section B") via `POST /api/classes/{classId}/sections`. Each section belongs to exactly one class (BR-ACA-05). Sections are where student enrollments and attendance are tracked.

---

## FR-ACA-06

**FR-ACA-06** allows School Admin to create subjects under a class (e.g., "Mathematics" for Class 6) via `POST /api/classes/{classId}/subjects`. Subject codes must be unique within a class (BR-ACA-07). Subject types include `MANDATORY` and `OPTIONAL`. These subjects are later referenced when configuring exams and marks.

---

## FR-ACA-07

**FR-ACA-07** allows School Admin to set an academic year as the current year via `PATCH /api/academic-years/{id}` with `{ "is_current": true }`. The system automatically unmarks the previously current year, ensuring exactly one current year per tenant at all times.

---

## FR-ACA-08

**FR-ACA-08** provides GET list endpoints for all academic entities (academic years, classes, sections, subjects). These are used to populate dropdown selectors throughout the UI — e.g., selecting a class when taking attendance or creating an exam.

---

## FR-SET-01

**FR-SET-01** allows School Admin to view their tenant's settings via `GET /api/settings`. The response includes the school logo URL, address, phone number, and email — all of which are used in report branding and communication.

---

## FR-SET-02

**FR-SET-02** allows School Admin to update school branding information via `PATCH /api/settings`. Fields include logo URL, address, phone, and email. Only the School Admin role can modify settings (BR-SET-01).

---

## FR-SET-03

**FR-SET-03** handles logo upload when the School Admin updates settings. When a logo file is uploaded, the system sends it to Cloudinary for storage and CDN delivery. The returned URL is saved in the `tenant_settings` record. Cloudinary handles image optimization and transformations.

---

## FR-UP-01

**FR-UP-01** allows School Admin to create Manager accounts via `POST /api/managers`. The School Admin provides the Manager's name, email, password, and phone. The user is created in the `users` table with `role = MANAGER` and `is_active = true`.

**Validation:**
- Email already in use → `409 EMAIL_EXISTS`

---

## FR-UP-02

**FR-UP-02** allows School Admin to activate or deactivate a Manager via `PATCH /api/managers/{id}`. Deactivation immediately invalidates all the Manager's active sessions by incrementing `token_version` (BR-UP-06), and the Manager cannot log in until reactivated (FR-AUTH-12).

---

## FR-UP-03

**FR-UP-03** provides the School Admin a list of all Managers with their active status, last login timestamp, and a summary of their assigned permissions via `GET /api/managers`. This is the main user management view.

---

## FR-UP-04

**FR-UP-04** allows School Admin to assign fine-grained action-level permissions to a Manager via `PUT /api/managers/{id}/permissions`. The School Admin specifies which modules the Manager can access and which actions (view, create, edit, delete, print, export) are allowed. Modules not listed are denied by default (BR-UP-05).

---

## FR-UP-05

**FR-UP-05** ensures that the School Admin can only delegate modules that were assigned by the Platform Admin. The `GET /api/permissions/available-modules` endpoint returns only the modules present in the tenant's `tenant_modules` table. This enforces the Tier 2 permission model boundary.

---

## FR-UP-06

**FR-UP-06** ensures immediate session invalidation when a Manager is deactivated. Deactivation increments the `token_version` field on the `users` record. All existing JWT tokens (which embed the token version at issuance) become invalid, forcing the Manager to re-authenticate if reactivated.

---

## FR-UP-07

**FR-UP-07** allows the School Admin to reset a Manager's password without requiring the current password. This is needed when a Manager forgets their password or is locked out.

**How it works:**

1. School Admin calls `PATCH /api/managers/{id}/reset-password` with `{ "new_password": "NewPass123" }`
2. System updates the Manager's `password_hash` in the `users` table
3. System increments `token_version` to invalidate all existing sessions (BR-AUTH-09)
4. Manager must log in again with the new password

**Validation:**
- Manager not found → `404 MANAGER_NOT_FOUND`
- Only School Admin of the same tenant → `403 FORBIDDEN`

---

## FR-MM-01

**FR-MM-01** allows School Admin to view all modules assigned by Platform Admin along with their current enabled/disabled status via `GET /api/tenant-modules`. This is the module management overview.

---

## FR-MM-02

**FR-MM-02** allows School Admin to toggle a module on or off via `PATCH /api/tenant-modules/{module}`. The School Admin can only toggle modules that were pre-assigned by the Platform Admin (BR-MM-02). Disabling a module hides it from all users in the tenant.

---

## FR-MM-03

**FR-MM-03** enforces module disablement at both the UI and API layers. When a module is toggled off, the sidebar hides its navigation items, and all API endpoints belonging to that module return `403 FORBIDDEN`. Existing data is preserved and restored when the module is re-enabled (BR-MM-04).

---

## FR-STU-01

**FR-STU-01** accepts admission applications from prospective students via `POST /api/applications`. The form collects applicant details including name, gender, date of birth, guardian contact, and address. The application is created with `status = PENDING` and awaits Manager review.

**What happens:**
- A unique application number is generated: `APP-{year}-{seq}` (FR-STU-02)
- The application is linked to the target academic year and applying class

---

## FR-STU-02

**FR-STU-02** generates a unique application number for each admission application. The format is `APP-{year}-{seq}` (e.g., `APP-2026-000001`), where `year` is the 4-digit admission year and `seq` is a zero-padded sequential counter per tenant. This number is the application's reference identifier throughout the admission process.

---

## FR-STU-03

**FR-STU-03** allows Managers with admission permission to view pending applications via `GET /api/applications?status=PENDING`. Results are paginated and filterable. This is the admission review queue where Managers can approve or reject applicants.

---

## FR-STU-04

**FR-STU-04** approves a pending application via `POST /api/applications/{id}/approve`, which creates a student record and an enrollment in a single atomic transaction (BR-STU-06). The Manager specifies the section to enroll the student in. On approval:
1. `students` row created with auto-generated registration number
2. `student_enrollments` row created for the specified section and academic year
3. Application status updated to `APPROVED`

**Validation:**
- Application already reviewed → `409 ALREADY_REVIEWED`
- Invalid section → `422`

---

## FR-STU-05

**FR-STU-05** rejects a pending application via `POST /api/applications/{id}/reject`. The Manager provides a rejection reason. The application status is updated to `REJECTED`. No student record is created.

**Validation:**
- Application already reviewed → `409 ALREADY_REVIEWED`

---

## FR-STU-06

**FR-STU-06** auto-generates the registration number during application approval. The format is `YY` (2-digit admission year) + continuous sequence per tenant (e.g., the first student admitted in 2026 gets `26000001`). The sequence is generated atomically using `SELECT ... FOR UPDATE` to prevent race conditions (BR-STU-02). The registration number is permanent and never changes (BR-STU-03).

---

## FR-STU-07

**FR-STU-07** auto-assigns a roll number when a student is enrolled in a section. Roll numbers are sequential within each (section + academic year) combination (BR-STU-04). The system assigns the next available number (e.g., if the section already has 30 students, the next student gets roll number 31).

---

## FR-STU-08

**FR-STU-08** provides full CRUD operations on student personal information via `/api/students` endpoints. School Admin and authorized Managers can view, create, update, and search student records. This covers changes to name, address, guardian details, and other personal fields.

---

## FR-STU-09

**FR-STU-09** provides student search by name, roll number, registration number, or class via `GET /api/students?search=...`. Additional filters include `class_id` and `academic_year_id`. Results include the student's current enrollment details.

---

## FR-STU-10

**FR-STU-10** allows School Admin to promote students to the next class, either individually or in bulk via `POST /api/students/promote`. The request specifies the from-class, to-class, section, and list of student IDs. New enrollments are created for the next academic year with fresh roll numbers. The operation is transactional — all-or-nothing (BR-STU-10 edge case).

---

## FR-STU-11

**FR-STU-11** handles end-of-year outcomes for students via `POST /api/students/{id}/outcome`. Supported outcomes:
- **Promote** — moves to next class
- **Repeat** — stays in same class
- **Graduate** — marks as graduated
- **Transfer** — marks as transferred out
- **Dropout** — marks as dropped out

Each outcome updates the student's status and enrollment record accordingly.

---

## FR-STU-12

**FR-STU-12** allows School Admin to import students from an Excel file via `POST /api/students/import`. The system parses the file, validates rows, and creates student + enrollment records. Validation errors are reported per row without rolling back successful imports (partial success model).

---

## FR-STU-13

**FR-STU-13** allows School Admin to export student lists to Excel or PDF via `GET /api/students/export`. The export includes all student details and their current enrollment information, filtered by the provided query parameters.

---

## FR-ATT-01

**FR-ATT-01** initializes an attendance session by loading enrolled students for a given class, section, and date via `GET /api/attendance/sessions/init`. The response includes all students enrolled in that section for the current academic year, each defaulted to `PRESENT`. The user can then modify individual attendance statuses before saving.

---

## FR-ATT-02

**FR-ATT-02** defaults all students to `PRESENT` when initializing an attendance session. This follows the common-sense assumption that most students are present — the Manager only needs to mark exceptions (absent, late, leave).

---

## FR-ATT-03

**FR-ATT-03** saves the attendance session with all student records via `POST /api/attendance/sessions`. The session is created in `DRAFT` status, allowing further edits before final submission.

**Validation:**
- Duplicate date for same section → `409` (BR-ATT-01)

---

## FR-ATT-04

**FR-ATT-04** submits a draft attendance session, transitioning it from `DRAFT` to `SUBMITTED` via `POST /api/attendance/sessions/{id}/submit`. Once submitted, the session is considered final and corrections require explicit editing (FR-ATT-05) with audit logging.

**Validation:**
- Already submitted → `409 ALREADY_SUBMITTED`

---

## FR-ATT-05

**FR-ATT-05** allows editing attendance records after submission. Changes are tracked in `audit_logs` with old and new values. This provides an audit trail for attendance corrections. Edits are only possible for users with the appropriate permission.

---

## FR-ATT-06

**FR-ATT-06** sends SMS notifications to guardians of absent students after an attendance session is submitted. Each SMS deducts 1 credit from the tenant's `sms_balance`. If balance is zero, SMS sending is blocked but the session status is unaffected (BR-NOT-01).

---

## FR-ATT-07

**FR-ATT-07** provides attendance reports for School Admin, filterable by class, student, and date range. The reports aggregate attendance data across sessions and can be used for monitoring student attendance patterns.

---

## FR-RES-01

**FR-RES-01** allows School Admin to create exams for a specific class via `POST /api/exams`. The exam includes a name, start/end dates, and is linked to an academic year and class. The exam starts in `DRAFT` status.

**Validation:**
- Duplicate exam name for the same class and academic year → `409` (BR-RES-01)

---

## FR-RES-02

**FR-RES-02** configures the subjects that are part of an exam via `POST /api/exams/{id}/subjects`. For each subject, the School Admin specifies full marks, pass marks, and display order. Subjects are drawn from the class's subject pool. Configuration is only possible while the exam is in `DRAFT` status.

---

## FR-RES-03

**FR-RES-03** allows School Admin to create grade scales via `POST /api/grade-scales`. Grade scales define the mapping from mark ranges to letter grades and GPA (e.g., 80-100 → A+ → 5.00). Grade scales are tenant-wide (see Q-RES-03 for per-year considerations).

---

## FR-RES-04

**FR-RES-04** allows Manager to enter marks per student per subject via `PUT /api/exams/{id}/marks`. The Manager provides obtained marks for each student-subject pair. If a student was absent, `is_absent = true` is set and marks must be null (BR-RES-06). Note: marks should be entered while the exam is in DRAFT — the precondition "Exam is PUBLISHED" is a known issue in the SRS.

**Validation:**
- Absent student with marks → validation error

---

## FR-RES-05

**FR-RES-05** calculates total marks, percentage, letter grade, and GPA at runtime based on the entered marks and the tenant's grade scale. These values are computed on display — they are never stored in the `marks` table (BR-RES-05), ensuring they always reflect the latest grade scale configuration.

---

## FR-RES-06

**FR-RES-06** publishes exam results via `POST /api/exams/{id}/publish`. Only School Admin can publish (BR-RES-03). Publishing sets `exams.result_published_at` and automatically triggers SMS/Email notifications to guardians. Publishing is a one-way gate — editing requires unpublishing first.

**Validation:**
- Marks missing for some students → `400 MARKS_INCOMPLETE`
- Non-School Admin → `403 FORBIDDEN`

---

## FR-RES-07

**FR-RES-07** unpublishes published results via `POST /api/exams/{id}/unpublish`, allowing marks to be edited. This returns the exam to a state where marks can be modified. After editing, the exam must be published again for the changes to be visible.

---

## FR-RES-08

**FR-RES-08** generates a rank list for the exam via `GET /api/exams/{id}/rank-list`. Students are ranked by total marks in descending order. Ranking is only available after results are published.

---

## FR-NTC-01

**FR-NTC-01** allows authorized users (School Admin or Manager with permission) to upload a notice as a PDF or image file via `POST /api/notices`. The file is uploaded to Cloudinary and the notice record stores the file URL. Notices can be scheduled for future publication (FR-NTC-02).

---

## FR-NTC-02

**FR-NTC-02** allows notices to be scheduled for future publication. When a notice is created with a `publish_at` timestamp set in the future, it remains unpublished (`is_published = false`) until the scheduled time is reached.

---

## FR-NTC-03

**FR-NTC-03** auto-publishes scheduled notices when their `publish_at` timestamp is reached. This is implemented via a cron job or queue worker that periodically checks for due notices and sets `is_published = true`.

---

## FR-NTC-04

**FR-NTC-04** archives a published notice via `POST /api/notices/{id}/archive`. Archived notices are hidden from the regular notice list but preserved for record-keeping. Published notices cannot be edited — they must be archived and a new notice created (BR-NTC-01).

---

## FR-NTC-05

**FR-NTC-05** allows all authenticated users within the tenant to view published notices via `GET /api/notices`. Only notices with `is_published = true` and `publish_at <= now()` are returned. Expired notices (past `expires_at`) are excluded.

---

## FR-NTC-06

**FR-NTC-06** auto-hides expired notices when their `expires_at` date is reached. Like auto-publishing, this is implemented via a cron job or queue worker. Expired notices are not deleted — they remain in the database but are excluded from query results.

---

## FR-NOT-01

**FR-NOT-01** sends SMS notifications to guardians of absent students when the Manager triggers it via `POST /api/attendance/sessions/{id}/send-sms`. Messages are sent through the centralized provider. Each SMS consumes 1 credit from the tenant's balance.

---

## FR-NOT-02

**FR-NOT-02** automatically sends SMS/Email notifications to guardians when exam results are published. This is triggered by the result publish action (FR-RES-06). The notification includes the student's grades or a link to view the results.

---

## FR-NOT-03

**FR-NOT-03** sends password reset emails when a user requests a password reset via the forgot-password flow. This is a system-generated notification with a time-limited reset link.

---

## FR-NOT-04

**FR-NOT-04** allows authorized users to send ad-hoc SMS or Email messages via `POST /api/notifications/send`. The user specifies the channel (SMS/Email), recipient, and message content. This is useful for emergency communications or custom alerts.

---

## FR-NOT-05

**FR-NOT-05** deducts 1 SMS credit from `tenants.sms_balance` each time an SMS is sent successfully. The deduction happens after the provider confirms delivery, ensuring accurate balance tracking.

---

## FR-NOT-06

**FR-NOT-06** blocks SMS sending when the tenant's SMS balance reaches zero. Email notifications continue to work. The endpoint returns `402 SMS_QUOTA_EXCEEDED`. The Platform Admin can increase the balance via FR-PLT-09.

---

## FR-NOT-07

**FR-NOT-07** logs every notification attempt (SMS and Email) in the `notifications` table, regardless of success or failure. Each log entry includes the channel, recipient, message, status, trigger type, and provider response. Logs are immutable once written (BR-NOT-04).

---

## FR-NOT-08

**FR-NOT-08** retries failed notifications via a cron or queue job. Retries are configurable (max attempts, interval). Each retry increments `retry_count` and updates the log status. After exhausting retries, the notification is marked as permanently failed.

---

## FR-RPT-01

**FR-RPT-01** generates a printable student ID card PDF via `GET /api/reports/students/{id}/id-card`. The card includes the student's photo, name, registration number, class, section, roll number, and school branding (logo + name from Settings). The PDF is streamed as a binary response.

---

## FR-RPT-02

**FR-RPT-02** generates a Transfer Certificate (TC) PDF for students with status `TRANSFERRED`. The TC includes student details, academic history, and the school's official seal and branding.

---

## FR-RPT-03

**FR-RPT-03** generates attendance reports in PDF or Excel format, filterable by class and academic year. Reports include attendance summaries (total days, present, absent, percentage) per student.

---

## FR-RPT-04

**FR-RPT-04** generates result report PDFs (per student or per exam). Per-student reports show all subjects and grades. Per-exam reports show all students' marks for a specific exam. Reports are generated only for published results.

---

## FR-RPT-05

**FR-RPT-05** ensures all generated reports include the school's branding (logo and name) from `tenant_settings`. This guarantees consistent, professional-looking documents across all report types.

---

## FR-DSH-01

**FR-DSH-01** displays the School Admin dashboard at `GET /api/dashboard/school-admin` with key metrics: total students, today's attendance percentage, whether today's attendance has been taken, recent notices, SMS balance/quota, and quick action links. All metrics are scoped to the current academic year (BR-DSH-01).

---

## FR-DSH-02

**FR-DSH-02** displays the Manager dashboard at `GET /api/dashboard/manager` with a subset of widgets based on the Manager's assigned permissions (BR-DSH-02). A Manager with only attendance access sees attendance widgets; a Manager with student management access sees student widgets.

---

## FR-DSH-03

**FR-DSH-03** scopes all dashboard metrics to the current academic year. This ensures consistency — counts, percentages, and lists all reflect the active academic year context.

---

## FR-AUD-01

**FR-AUD-01** logs all Create, Update, and Delete operations on student data in `audit_logs`. Each log entry captures the actor (user), action, entity type, old/new values (as JSONB), IP address, and timestamp. Routine reads and simple queries are NOT logged (BR-AUD-02).

---

## FR-AUD-02

**FR-AUD-02** logs attendance corrections made after a session has been submitted. The log captures both the previous and updated attendance statuses, providing an audit trail for any post-submission changes.

---

## FR-AUD-03

**FR-AUD-03** logs result publish and unpublish actions. This provides accountability for when results were made available and who performed the action.

---

## FR-AUD-04

**FR-AUD-04** logs all Manager permission changes. When a School Admin updates a Manager's permissions, the old and new permission sets are recorded with the actor's details.

---

## FR-AUD-05

**FR-AUD-05** logs SMS balance adjustments. When the Platform Admin modifies a tenant's SMS balance, the adjustment amount, previous balance, and new balance are recorded.

---

## FR-AUD-06

**FR-AUD-06** logs tenant activation and deactivation events. The log captures who performed the action and the timestamp, providing an audit trail for tenant lifecycle changes.

---

## FR-AUD-07

**FR-AUD-07** allows School Admin to view audit logs for their own tenant via `GET /api/audit-logs`. Logs are filterable by module, action type, and date range. This provides transparency for internal operations within the school.

---

## FR-AUD-08

**FR-AUD-08** allows Platform Admin to view audit logs across all tenants via `GET /api/admin/audit-logs`. This enables centralized monitoring and compliance auditing across the entire platform.
