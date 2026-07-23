# FR → OpenAPI Path → Prisma Model — Traceability Matrix

> **Source:** SRS Appendix A (Endpoints), SRS Appendix C (RTM), DB Dictionary
> **Total:** 14 modules, 80 endpoints, 23 Prisma tables

## Module 1: Platform & Multi-Tenant Foundation

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-PLT-01 | POST | /api/v1/admin/tenants | 01-platform.yaml | tenants, users, tenant_settings, tenant_modules |
| FR-PLT-02 | (validation) | — | 01-platform.yaml | tenants (subdomain unique) |
| FR-PLT-03 | (side-effect) | — | 01-platform.yaml | users |
| FR-PLT-04 | (side-effect) | — | 01-platform.yaml | tenant_settings |
| FR-PLT-05 | (side-effect) | — | 01-platform.yaml | tenant_modules |
| FR-PLT-06 | PATCH | /api/v1/admin/tenants/{id} | 01-platform.yaml | tenants |
| FR-PLT-07 | (middleware) | — | — | tenants (is_active check) |
| FR-PLT-08 | GET | /api/v1/admin/tenants | 01-platform.yaml | tenants |
| FR-PLT-09 | PATCH | /api/v1/admin/tenants/{id}/sms-balance | 01-platform.yaml | tenants |
| FR-PLT-10 | GET | /api/v1/admin/notifications | 01-platform.yaml | notifications |
| FR-PLT-11 | PATCH | /api/v1/admin/users/{id}/reset-password | 01-platform.yaml | users |
| FR-PLT-12 | GET | /api/v1/admin/analytics | 01-platform.yaml | tenants, students, etc. |
| FR-PLT-13 | PATCH | /api/v1/admin/settings | 01-platform.yaml | (platform config) |

## Module 2: Authentication

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-AUTH-01 | POST | /api/v1/admin/auth/login | 02-auth.yaml | platform_admins |
| FR-AUTH-02 | POST | /api/v1/auth/login | 02-auth.yaml | users, tenants |
| FR-AUTH-06 | POST | /api/v1/auth/logout | 02-auth.yaml | refresh_tokens |
| FR-AUTH-07 | POST | /api/v1/auth/refresh | 02-auth.yaml | refresh_tokens |
| FR-AUTH-08 | POST | /api/v1/auth/change-password | 02-auth.yaml | users, platform_admins |
| FR-AUTH-09 | GET | /api/v1/auth/me | 02-auth.yaml | users, platform_admins |

## Module 3: Academic Structure

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-ACA-01 | POST | /api/v1/academic-years | 03-academic.yaml | academic_years |
| FR-ACA-04 | POST | /api/v1/classes | 03-academic.yaml | classes |
| FR-ACA-05 | POST | /api/v1/classes/{classId}/shifts | 03-academic.yaml | shifts |
| FR-ACA-06 | POST | /api/v1/shifts/{shiftId}/sections | 03-academic.yaml | sections |
| FR-ACA-07 | POST | /api/v1/classes/{classId}/subjects | 03-academic.yaml | subjects |
| FR-ACA-08 | PATCH | /api/v1/academic-years/{id} | 03-academic.yaml | academic_years |
| FR-ACA-09 | GET | /api/v1/academic-years, /api/v1/classes/{classId}/shifts | 03-academic.yaml | academic_years, shifts |
| FR-ACA-10 | POST | /api/v1/academic-years/{id}/close, /api/v1/academic-years/{id}/reopen | 03-academic.yaml | academic_years |

## Module 4: Settings

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-SET-01 | GET | /api/v1/settings | 04-settings.yaml | tenant_settings |
| FR-SET-02 | PATCH | /api/v1/settings | 04-settings.yaml | tenant_settings |
| FR-NOT-09 | PATCH | /api/v1/settings/notifications | 04-settings.yaml | tenant_settings |

## Module 5: User & Permission Management

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-UP-01 | POST | /api/v1/managers | 05-user-permission.yaml | users |
| FR-UP-02 | PATCH | /api/v1/managers/{id} | 05-user-permission.yaml | users |
| FR-UP-03 | GET | /api/v1/managers | 05-user-permission.yaml | users |
| FR-UP-04 | PUT | /api/v1/managers/{id}/permissions | 05-user-permission.yaml | manager_permissions |
| FR-UP-05 | GET | /api/v1/permissions/available-modules | 05-user-permission.yaml | tenant_modules |
| FR-UP-07 | PATCH | /api/v1/managers/{id}/reset-password | 05-user-permission.yaml | users |
| FR-UP-08 | PATCH | /api/v1/managers/{id}/profile | 05-user-permission.yaml | users |

## Module 6: Module Management

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-MM-01 | GET | /api/v1/tenant-modules | 06-module-management.yaml | tenant_modules |
| FR-MM-02 | PATCH | /api/v1/tenant-modules/{module} | 06-module-management.yaml | tenant_modules |

## Module 7: Student Management & Admission

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-STU-01 | POST | /api/v1/applications | 07-student.yaml | applications |
| FR-STU-03 | GET | /api/v1/applications | 07-student.yaml | applications |
| FR-STU-04 | POST | /api/v1/applications/{id}/approve | 07-student.yaml | applications, students, student_enrollments |
| FR-STU-05 | POST | /api/v1/applications/{id}/reject | 07-student.yaml | applications |
| FR-STU-08 | POST/GET/PATCH/DELETE | /api/v1/students, /api/v1/students/{id} | 07-student.yaml | students, student_enrollments |
| FR-STU-09 | GET | /api/v1/students (filtered) | 07-student.yaml | students |
| FR-STU-10 | POST | /api/v1/students/promote | 07-student.yaml | student_enrollments |
| FR-STU-11 | POST | /api/v1/students/{id}/outcome | 07-student.yaml | student_enrollments |
| FR-STU-12 | POST/GET | /api/v1/students/import, /api/v1/students/import/template | 07-student.yaml | students |
| FR-STU-13 | GET | /api/v1/students/export | 07-student.yaml | students |

## Module 8: Attendance Management

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-ATT-01 | GET | /api/v1/attendance/sessions/init | 08-attendance.yaml | attendance_sessions, attendance_records |
| FR-ATT-03 | POST | /api/v1/attendance/sessions | 08-attendance.yaml | attendance_sessions, attendance_records |
| FR-ATT-04 | POST | /api/v1/attendance/sessions/{id}/submit | 08-attendance.yaml | attendance_sessions |
| FR-ATT-05 | PATCH | /api/v1/attendance/sessions/{id}/records | 08-attendance.yaml | attendance_records |
| FR-ATT-06 | POST | /api/v1/attendance/sessions/{id}/send-sms | 08-attendance.yaml | notifications |
| FR-ATT-07 | GET | /api/v1/attendance/reports | 08-attendance.yaml | attendance_records |

## Module 9: Result Management

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-RES-01 | POST | /api/v1/exams | 09-result.yaml | exams |
| FR-RES-02 | POST | /api/v1/exams/{id}/subjects | 09-result.yaml | exam_subjects |
| FR-RES-03 | POST | /api/v1/grade-scales | 09-result.yaml | grade_scales |
| FR-RES-04 | PUT | /api/v1/exams/{id}/marks | 09-result.yaml | marks |
| FR-RES-06 | POST | /api/v1/exams/{id}/publish | 09-result.yaml | exams |
| FR-RES-07 | POST | /api/v1/exams/{id}/unpublish | 09-result.yaml | exams |
| FR-RES-08 | GET | /api/v1/exams/{id}/rank-list | 09-result.yaml | marks, exam_subjects |
| FR-RES-09 | GET | /api/v1/exams/{id}/results | 09-result.yaml | marks, exam_subjects |

## Module 10: Notice Board

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-NTC-01 | POST | /api/v1/notices | 10-notice.yaml | notices |
| FR-NTC-04 | POST | /api/v1/notices/{id}/archive | 10-notice.yaml | notices |
| FR-NTC-05 | GET | /api/v1/notices | 10-notice.yaml | notices |
| FR-NTC-07 | DELETE | /api/v1/notices/{id} | 10-notice.yaml | notices |

## Module 11: Notification System

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-NOT-04 | POST | /api/v1/notifications/send | 11-notification.yaml | notifications |
| FR-NOT-07 | GET | /api/v1/notifications | 11-notification.yaml | notifications |

## Module 12: Reports

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-RPT-01 | GET | /api/v1/reports/students/{id}/progress | 12-reports.yaml | (read-only: marks, attendance) |
| FR-RPT-02 | GET | /api/v1/reports/students/{id}/tc | 12-reports.yaml | (read-only: students, enrollments) |
| FR-RPT-03 | GET | /api/v1/reports/attendance | 12-reports.yaml | (read-only: attendance_records) |
| FR-RPT-04 | GET | /api/v1/reports/results | 12-reports.yaml | (read-only: marks, exam_subjects) |
| FR-RPT-06 | GET | /api/v1/reports/load | 12-reports.yaml | (read-only: all student data) |

## Module 13: Dashboard

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-DSH-01 | GET | /api/v1/dashboard/school-admin | 13-dashboard.yaml | (aggregated: students, attendance, notices) |
| FR-DSH-02 | GET | /api/v1/dashboard/manager | 13-dashboard.yaml | (aggregated: scoped by permissions) |
| FR-DSH-04 | GET | /api/v1/admin/dashboard | 13-dashboard.yaml | (aggregated: all tenants) |

## Module 14: Audit Logging

| FR ID | Method | Path | YAML File | Prisma Models |
|-------|--------|------|-----------|---------------|
| FR-AUD-07 | GET | /api/v1/audit-logs | 14-audit.yaml | audit_logs |
| FR-AUD-08 | GET | /api/v1/admin/audit-logs | 14-audit.yaml | audit_logs |
