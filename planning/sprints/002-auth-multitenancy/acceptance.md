# Acceptance Criteria — Sprint 002: Auth & Multi-Tenancy

**Linked to:** requirements.md  
**Last updated:** 2026-05-29

A feature is done when ALL its criteria pass.

---

## AC-001: School Creation (Super Admin)

**Given** a logged-in Super Admin on `/admin/schools/new`  
**When** they submit a valid school creation form (name, curriculum, county, headteacher email)  
**Then:**
- A Clerk organisation is created with `orgId` stored in `schools.clerkOrgId`
- A `schools` row exists with correct name, slug, curriculumType, subscriptionTier=STANDARD, subscriptionStatus=TRIAL
- An invite email is sent to the headteacher email via Clerk
- AuditLog entry: `{ action: "school.created", entityType: "School", entityId: schoolId }`
- Super Admin sees the new school in the schools list

**Test data:** name="St Mary's Academy", county="Nairobi", curriculum=CBC  
**Test type:** Integration (Clerk test mode)

**Given** the same school name is submitted again  
**When** the form is submitted  
**Then** a 409 error is shown: "A school with this name already exists"

---

## AC-002: Onboarding Wizard

**Given** a Headteacher who has just accepted their Clerk invite  
**When** they log in for the first time  
**Then** they are redirected to `/[schoolSlug]/onboarding`

**Given** the Headteacher is on step 1  
**When** they submit school profile data  
**Then** `OnboardingState.currentStep` advances to 2, data saved to `stepData`

**Given** the Headteacher closes the browser mid-wizard (step 3 of 5)  
**When** they log in again  
**Then** they land back on step 3 with previously entered data pre-filled

**Given** all 5 steps are complete  
**When** step 5 is submitted  
**Then** `schools.onboardingComplete = true` and user redirected to `/[schoolSlug]/dashboard`

**Test type:** E2E (Playwright)

---

## AC-003: Role Definitions

**Given** a user is created with role=ACCOUNTANT  
**When** they log in and visit `/[schoolSlug]/academic/exams`  
**Then** they receive a 403 redirect to `/403`

**Given** a user with role=DEAN_ACADEMICS  
**When** they visit `/[schoolSlug]/academic/exams`  
**Then** they see the exams management page

**Test data:** Two users — one ACCOUNTANT, one DEAN_ACADEMICS, same school  
**Test type:** Integration

All 16 roles defined in `UserRole` Prisma enum. All 16 have a corresponding dashboard layout shell (even if empty). No user can access a route outside their role's allowlist.

---

## AC-004: Unique ID Generation

**Given** a new user is created via `school.inviteUser`  
**When** the user record is inserted  
**Then** `users.uniqueId` = `EDU-[SCHOOL_CODE]-[ROLE_CODE]-[PADDED_SEQUENCE]`

Example: 3rd student created at school with code "STM001" = `EDU-STM001-STU-00003`

**Given** two users are created simultaneously  
**When** both inserts complete  
**Then** no two users share the same uniqueId (sequence must be atomic)

**Test type:** Unit (sequence generation function) + Integration

---

## AC-005: Row-Level Security

**Given** User A belongs to School A (schoolId=A)  
**When** User A's authenticated tRPC context queries `users` table  
**Then** only users where `school_id = A` are returned — zero rows from School B

**Given** a direct SQL query is attempted without `app.current_school_id` set  
**When** the query runs  
**Then** it returns 0 rows (RLS default-deny)

**Test type:** Integration (direct DB test against seeded multi-school data)

RLS policies must be present on: `schools`, `users`, `classes`, `terms`, `audit_logs`, `onboarding_states`

---

## AC-006: School Email Enforcement

**Given** a school with emailDomain="stmarys.ac.ke"  
**When** a staff invite is sent to "teacher@gmail.com"  
**Then** the invite is rejected with: "Staff must use a school-assigned email (@stmarys.ac.ke)"

**Given** a PARENT role invite  
**When** sent to "parent@gmail.com"  
**Then** the invite succeeds (parents exempt from domain restriction)

**Test type:** Unit

---

## AC-007: Role-Based Dashboard Routing

**Given** any of the 16 roles logs in  
**When** they land on `/[schoolSlug]/dashboard`  
**Then** they see the correct dashboard shell for their role (verified by page heading + nav items)

**Given** an unauthenticated user visits `/[schoolSlug]/dashboard`  
**When** Next.js middleware runs  
**Then** they are redirected to `/sign-in`

**Test type:** E2E (Playwright — one test per role group: admin, teacher, financial, operational, parent, student)

---

## Regression Baseline

Sprint 002 is the first sprint. No prior features to protect.

---

## Definition of Done

- All AC-001 through AC-007 pass
- RLS policies verified by automated integration test
- All 16 UserRole enum values have a corresponding dashboard route and layout
- No `// ARCH-QUESTION:` comments unresolved
- AuditLog entries verified for school.created and user.created
- CONTEXT.md updated with Sprint 002 complete status
- STATE.md updated
