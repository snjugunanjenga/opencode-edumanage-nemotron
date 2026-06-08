# Requirements — Sprint 002: Auth & Multi-Tenancy

**Project:** EduManage  
**Sprint:** 002  
**Status:** Approved  
**Last updated:** 2026-05-29  
**Architect sign-off:** Approved

---

## Sprint Goal

Establish the complete authentication, multi-tenancy, and role system so that every subsequent sprint can build on a secure, school-isolated foundation — and the first school can be onboarded end-to-end.

---

## Functional Requirements

### FR-001: Clerk Multi-Tenant Setup

**Priority:** P0  
**User story:** As a Super Admin, I want to create a new school on the platform so that the school gets an isolated tenant with its own users and data.  
**Acceptance criteria ref:** AC-001

Clerk Organisation is created for each school. `orgId` = `schoolId`. School record inserted in `schools` table with name, slug, curriculum type, subscription tier. Super Admin is the platform-level admin (not part of any school org).

**Edge cases:**
- Duplicate school slug: return 409 with friendly message
- Clerk org creation fails: rollback DB insert, return 500

---

### FR-002: School Onboarding Wizard (Headteacher)

**Priority:** P0  
**User story:** As a Headteacher, I want to complete a setup wizard after my school is created so that the platform is configured for my school before staff are invited.  
**Acceptance criteria ref:** AC-002

5-step wizard: (1) School profile (name, logo, county, curriculum type), (2) Academic year + term dates, (3) Create classes/streams, (4) Invite first staff members (email + role), (5) Confirmation + dashboard redirect. Wizard state persisted so it can be resumed if interrupted.

**Edge cases:**
- Logo upload fails: wizard continues without logo
- Staff invite email bounces: flagged in UI, can retry

---

### FR-003: All 16 Role Definitions

**Priority:** P0  
**User story:** As a platform, I need to recognise all 16 user roles and enforce their permissions so that each user only sees and does what their role allows.  
**Acceptance criteria ref:** AC-003

Roles defined as Clerk `publicMetadata.role` enum:
`SUPER_ADMIN | HEADTEACHER | DEAN_ACADEMICS | CLASS_TEACHER | SUBJECT_TEACHER | ACCOUNTANT | PROCUREMENT_OFFICER | IT_SUPPORT | HR_MANAGER | TRANSPORT_MANAGER | LIBRARIAN | SCHOOL_NURSE | CATERESS | MATRON | PARENT | STUDENT`

Each role has: allowed routes (middleware), allowed tRPC procedures (protectedProcedure), dashboard layout variant.

**Edge cases:**
- User with no role: redirect to /error/no-role
- Role change by Admin: Clerk metadata updated + DB updated atomically

---

### FR-004: Unique ID Generation

**Priority:** P0  
**User story:** As a school admin, I want every user to receive a unique platform ID at registration so that users are uniquely identifiable across the system.  
**Acceptance criteria ref:** AC-004

Format: `EDU-[SCHOOL_CODE]-[ROLE_CODE]-[5_DIGIT_SEQUENCE]`  
Example: `EDU-STR001-STU-00234`  
Generated server-side at user creation. Stored in `users.unique_id`. Displayed on user profile + printed ID cards (future).

---

### FR-005: Row-Level Security Policies

**Priority:** P0  
**User story:** As the platform, I need every database query to be automatically scoped to the current school so that school data is never mixed.  
**Acceptance criteria ref:** AC-005

RLS enabled on all tables. Policy template applied. Prisma client wrapper sets `app.current_school_id` from tRPC context on every query. Integration test: authenticated as School A user, attempt to query School B data — must return 0 rows.

---

### FR-006: School-Assigned Email Enforcement

**Priority:** P1  
**User story:** As a Headteacher, I want all staff and student logins to use school-assigned email addresses so that access is controlled and revocable.  
**Acceptance criteria ref:** AC-006

Clerk allowlist configured per org: only `*@[school_domain]` emails allowed for STAFF and STUDENT roles. PARENT role allows any email. Email domain set during school onboarding wizard step 1.

**Edge cases:**
- School doesn't have a domain yet: allow any email temporarily, flag in setup
- Staff member leaves: Headteacher deactivates user → Clerk org membership removed → DB `isActive = false`

---

### FR-007: Role-Based Dashboard Routing

**Priority:** P0  
**User story:** As any user, I want to land on my role-specific dashboard after login so that I immediately see what's relevant to my role.  
**Acceptance criteria ref:** AC-007

Post-login redirect: `/[schoolSlug]/dashboard` which renders the correct layout based on `role` from Clerk JWT. Each of the 16 roles has a distinct dashboard layout. Non-authenticated: redirect to `/sign-in`.

---

## Non-Functional Requirements

### NFR-001: Performance
All dashboard initial loads under 2 seconds on a standard Kenyan 4G connection (10Mbps). Server components used for data-heavy pages.

### NFR-002: Security
Every tRPC procedure wrapped in `protectedProcedure`. Middleware validates JWT on every request. RLS enforced at DB level as second layer.

### NFR-003: Audit
Every user creation, role change, and deactivation logged to `AuditLog`.

---

## Out of Scope (this sprint)

- Student/parent portal UI (Sprint 003+)
- Fee payment flows (Sprint 004)
- Any academic features (Sprint 003)
- Email notification templates (Sprint 005)
- Biometric integration
- SSO / Google Workspace login

---

## Open Questions (blocking this sprint)

| # | Question | Owner | Status |
|---|---|---|---|
| 1 | School email domain — required at onboarding or optional? | Operator | Open (see Q-006) |
