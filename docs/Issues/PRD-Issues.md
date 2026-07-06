# PRD Review Report

**Reviewer:** AI Code Review  
**Date:** July 6, 2026  
**Document:** `docs/PRD.md` v1.0 (Draft)

---

## Overall

The PRD is well-structured and comprehensive. The vision is clear, the two-tier permission model is elegant, and the business rules are well-documented. The gaps are mostly at the edges — the core is solid and ready for implementation. Below is the conversation-style review.

---

## Strengths

**Q:** What does the PRD do well?

- **Well-structured & comprehensive** — Covers vision, scope, roles, modules, business rules, and milestones clearly.
- **Two-tier permission model is elegant** — Module-level for Platform Admin → School Admin, action-level for School Admin → Manager. Simple yet powerful.
- **Registration number design is smart** — `YY + continuous sequence` without embedding class/section means the ID stays valid forever. Roll number as a separate mutable field on enrollment is the right call.
- **Tenant isolation via `tenant_id` on every table** with middleware enforcement is pragmatic for MVP while keeping the door open for DB-per-tenant later.
- **Centralized notification gateway** rather than per-tenant SMTP/SMS config is a good SaaS decision.
- **Business rules are clearly documented** (31 explicit rules in §9) — this removes ambiguity during implementation.

---

## Gaps & Ambiguities

**Q:** Public admission form without a portal (§6.3)

The admission workflow says *"Student fills admission form"* but the Out-of-Scope section excludes a parent/student portal. How does the student access this form? Is there a public-facing endpoint (e.g., `schoolname.sirius-skool.com/admissions`) that requires no login? This needs clarification before implementation.

**Recommendation:** Add a note that the admission form is a public route (no auth required) within the tenant's subdomain, and specify any anti-bot measures (CAPTCHA, rate limiting).

---

**Q:** Subject-level permissions for Results (§6.5)

The PRD says *"Marks entry can be done by multiple managers (e.g., subject-wise) if permissions allow"* but the permission model only supports module + action granularity. A Manager either gets "Enter Marks" on all subjects in Results or none. If subject-level delegation is intended, the permission model needs an additional dimension.

**Recommendation:** Call out subject-level permissions as a post-MVP enhancement, or if intended for MVP, add a `subject_id` or `exam_id` dimension to the permission model.

---

**Q:** Email quota (§6.7)

SMS has a well-defined quota system, but email is unmentioned. Unlimited email could be abused. Even a soft cap or a simple statement that it's not quota-limited would help.

**Recommendation:** Add email usage limits (even if generous) or explicitly state that email is not throttled.

---

**Q:** Audit Logging (§7)

Listed as in-scope in the cross-cutting section and mentioned briefly in Non-Functional Requirements (§10), but there's no functional description of the audit log UI — who can view it, what filters exist, how far back it goes.

**Recommendation:** Add a dedicated section in §7 (Cross-Cutting) for Audit Logging covering: what's logged, who can view logs, retention period, and any UI for searching/filtering. Define the data model minimally:

```
audit_logs {
  id, tenant_id, user_id, action, module,
  record_id, old_values, new_values,
  ip_address, user_agent, created_at
}
```

---

**Q:** File upload details

Cloudinary is named in §10 but there's no functional spec on file uploads: size limits, allowed types (image only? PDFs for notices?), who can upload, where uploads appear in the UI.

**Recommendation:** Add a subsection in §6.9 (Settings) or §10 specifying upload constraints.

---

**Q:** Platform Admin UI

The PRD describes what Platform Admin *can do* (create tenants, set quotas, etc.) but there's no dedicated section for the Platform Admin dashboard, tenant listing page, or management UI.

**Recommendation:** Add a brief section describing the Platform Admin interface — at minimum a tenant list, tenant detail/edit page, and quota management UI.

---

**Q:** Data migration / onboarding

Mentioned in passing (registration sequence starting value), but no spec for importing existing students, classes, attendance, or results from legacy systems. For a product targeting schools with existing data, this is a notable gap.

**Recommendation:** Add a data migration section covering: import templates, bulk CSV/Excel import for students, validation rules, and rollback on failure.

---

**Q:** Subdomain detection at login (§5)

*"Login page detects the subdomain"* — how exactly? Via `window.location.hostname` parsing? What about localhost development? A fallback mechanism (manual tenant selection or tenant code entry) would be needed.

**Recommendation:** Document the subdomain detection mechanism and include a fallback for local/dev environments.

---

**Q:** Search/autocomplete

Student search is mentioned but no spec for global search or autocomplete behavior. This is minor for MVP but worth noting.

**Recommendation:** Add a brief note on search requirements (debounce, minimum characters, searchable fields).

---

## Inconsistencies

**Q:** Academic Year Management location

Academic Year Management appears in both §6.9 (Settings) and §7.3 (Cross-Cutting). Its features are split across two locations, making it easy to miss rules.

**Recommendation:** Keep the feature details in one place (e.g., §7.3) and reference it from §6.9.

---

**Q:** Dashboard quick actions vs permissions (§6.2)

Dashboard quick actions list *"Add Student, Take Attendance, Create Notice"* but these should be permission-dependent for Managers. The rule says *"Managers only see data for modules they have View permission on"* but quick actions are Create operations, not View.

**Recommendation:** Clarify that quick actions are gated by the Manager's Create permission on that module, not View.

---

**Q:** Multi-Language — in-scope or not?

Future Scope lists *"Multi-Language"* as medium priority, but §10 Non-Functional Requirements says *"Database schema should support multi-language labels from day one"*. This is an architectural decision that should either be explicitly in-scope or consciously deferred.

**Recommendation:** Either move multi-language schema support into in-scope and expand it, or remove the note from §10 and defer it entirely.

---

## Technical Considerations

**Q:** Permission evaluation performance

Action-level permissions per manager per module will require careful query design. Every page load needs to check `can(module, action)`. A cached permission bitmap per user (e.g., Redis hash) would be advisable from day one.

---

**Q:** Admission temp table TTL (§6.3)

*"auto-expire after 30 days (cron job or TTL index)"* — TTL indexes work in MongoDB but not in PostgreSQL/MySQL without a background job. A scheduled task or cron is needed unless using MongoDB.

---

**Q:** Registration number sequence atomicity

*"read counter → generate → increment → commit (all in one atomic transaction)"* — This needs a database-side atomic increment (e.g., `UPDATE ... SET seq = seq + 1 RETURNING seq` wrapped in a transaction, or a dedicated sequence table with `SELECT FOR UPDATE`).

---

**Q:** Session invalidation on deactivation (§7.1, Rule 14)

*"Deactivating a Manager immediately revokes access across all sessions"* — This requires either a token blacklist (Redis) or short-lived tokens with a `token_version` column checked on every request. Should be architected early.

---

**Q:** Subdomain wildcard SSL (§5)

Requires a wildcard cert for `*.sirius-skool.com` and infrastructure-level routing (reverse proxy / ingress) to direct traffic to the app with the subdomain available as a header.

---

## Recommendations

1. **Add a dedicated Audit Logging section** in §7 with data model and UI spec.
2. **Clarify the admission form access** — is it a public route? Add CAPTCHA/rate limiting note.
3. **Call out subject-level permissions** as post-MVP or expand the permission model.
4. **Add email usage limits** or explicitly state email is not throttled.
5. **Specify the tech stack** in §10 or a new Architecture section. The PRD is framework-agnostic which is fine, but decisions on backend (Laravel? Node.js? Django?), frontend (React? Vue?), and database (PostgreSQL? MySQL?) need to be made for implementation.
6. **Define the audit log data model** (suggested schema above).
7. **Resolve the Academic Year Management double-location** inconsistency.
8. **Clarify dashboard quick action permission gating**.

---

## Bottom Line

The PRD is **well-above average** in clarity and completeness. The core — roles, permissions, modules, business rules — is solid and ready for implementation. The gaps identified above are mostly at the edges. The biggest decision waiting is the **tech stack choice**, which will influence detailed schema design and project scaffolding.
