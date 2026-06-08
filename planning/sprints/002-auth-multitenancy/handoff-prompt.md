# Builder Handoff — Sprint 002: Auth & Multi-Tenancy

**Instructions:** Paste everything below this line into Claude Code.
Point Claude Code at the project root first.

---

You are a Builder agent implementing Sprint 002 of EduManage — a multi-tenant SaaS school management platform for Kenyan schools.

Your job is to implement — not redesign, not redefine scope.

## Before writing any code, read these files in order:

1. `AGENTS.md`
2. `CONTEXT.md`
3. `planning/STATE.md`
4. `planning/DECISIONS.md`
5. `planning/DOMAIN.md`
6. `planning/RISKS.md`
7. `planning/sprints/002-auth-multitenancy/requirements.md`
8. `planning/sprints/002-auth-multitenancy/blueprint.md`
9. `planning/sprints/002-auth-multitenancy/acceptance.md`

## Then summarise (before touching any code):

1. What this sprint accomplishes
2. Every file you expect to create or modify (with path)
3. The migration(s) you will run
4. Tests you will write and how you'll run them
5. Any blockers, ambiguities, or `// ARCH-QUESTION:` items

**Wait for operator approval of your summary before writing a single line of code.**

---

## Sprint Goal

Establish the complete authentication, multi-tenancy, and role system so that every subsequent sprint can build on a secure, school-isolated foundation — and the first school can be onboarded end-to-end.

---

## Stack

- Next.js 14 App Router + TypeScript strict
- Clerk (`@clerk/nextjs`) for auth
- Prisma 5 + Neon Postgres for database
- tRPC v11 for API layer
- Tailwind CSS + shadcn/ui for UI
- Zod for validation
- AWS S3 for logo upload (presigned POST)
- Inngest for async jobs (audit log writes)

---

## Implementation Sequence

### Phase 1: Project Bootstrap
1. Initialise Next.js 14 project with TypeScript strict, Tailwind, App Router
2. Install dependencies: `@clerk/nextjs`, `@prisma/client`, `prisma`, `@trpc/server`, `@trpc/client`, `@trpc/next`, `zod`, `@aws-sdk/client-s3`, `inngest`
3. Configure environment variables (see ENV section below)
4. Set up `prisma/schema.prisma` with School, User, OnboardingState, AuditLog models + all enums
5. Run `prisma migrate dev --name init_auth_tenancy`
6. Apply RLS policies (SQL migration file, not Prisma — use `prisma/migrations/manual/001_rls_policies.sql`)

### Phase 2: tRPC Foundation
7. Create `server/trpc.ts` — init tRPC, define `protectedProcedure(allowedRoles)` helper
8. Create `server/context.ts` — extract Clerk auth, inject `{ schoolId, userId, role }`, create `prismaWithRLS(schoolId)` wrapper
9. Create `server/db.ts` — Prisma client with RLS `SET LOCAL` wrapper
10. Create `app/api/trpc/[trpc]/route.ts` — tRPC HTTP handler

### Phase 3: School Router
11. Create `server/routers/school.ts` with procedures:
    - `school.create` (SUPER_ADMIN only)
    - `school.onboarding.saveStep`
    - `school.onboarding.getState`
    - `school.inviteUser`
    - `school.deactivateUser`
    - `school.listUsers`
12. Create `lib/unique-id.ts` — atomic unique ID generator
13. Create `lib/s3/presigned-upload.ts` — presigned POST for logo

### Phase 4: Clerk Webhook Handler
14. Create `app/api/webhooks/clerk/route.ts`
    - Handle `organizationMembership.created` → sync user to DB
    - Handle `user.deleted` → soft-delete in DB
    - Verify Svix signature

### Phase 5: Middleware
15. Create `middleware.ts` — Clerk `authMiddleware`:
    - Public routes: `/sign-in`, `/api/webhooks/*`
    - Protected routes: all others
    - Post-auth: redirect to `/onboarding` if `onboardingComplete=false` and role=HEADTEACHER
    - Role-based route guard (middleware allowlist per role group)

### Phase 6: UI — Onboarding Wizard
16. Create `app/(school)/[schoolSlug]/onboarding/page.tsx` — 5-step wizard
    - Step 1: School profile (name, logo upload, county, email domain)
    - Step 2: Academic year + 3 term date ranges
    - Step 3: Create initial classes (name, grade level, stream, curriculum)
    - Step 4: Invite staff (email, role, first name, last name — repeatable)
    - Step 5: Confirmation + "Go to Dashboard" CTA
    - Wizard state saved to DB on each step submit

### Phase 7: UI — Dashboard Shells
17. Create `app/(school)/[schoolSlug]/dashboard/page.tsx`
    - Server component reads role from Clerk JWT
    - Renders correct dashboard layout shell for role
    - 16 role variants — each shows: role name, school name, placeholder "Sprint 003 will add content here"
    - Shared nav sidebar with role-appropriate menu items (links only, no functionality yet)

### Phase 8: Super Admin UI
18. Create `app/(platform)/admin/schools/page.tsx` — school list
19. Create `app/(platform)/admin/schools/new/page.tsx` — create school form

### Phase 9: Tests
20. Write integration tests for RLS (seed 2 schools, verify cross-tenant query returns 0)
21. Write unit test for unique ID generator (concurrent generation, no collisions)
22. Write E2E test (Playwright): full onboarding flow for 1 school
23. Write E2E test: role-based routing (6 role variants, each hits forbidden route)

---

## Environment Variables Required

```env
# Clerk
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
CLERK_WEBHOOK_SECRET=
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/dashboard

# Database
DATABASE_URL=          # Neon connection string (pooled)
DIRECT_URL=            # Neon direct connection (for migrations)

# AWS S3
AWS_REGION=af-south-1
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_BUCKET=edumanage-assets

# Inngest
INNGEST_EVENT_KEY=
INNGEST_SIGNING_KEY=
```

---

## Critical Constraints

- Every table in `schema.prisma` MUST have `schoolId String` (except `School` itself and `AuditLog` which has it nullable)
- `SET LOCAL app.current_school_id` MUST be called inside every DB transaction via the `prismaWithRLS` wrapper
- RLS policies use `current_setting('app.current_school_id')::uuid` — test this works with Neon connection pooler in transaction mode
- Exam security: no exam tables in this sprint, but S3 bucket must be created with `block_public_access = true` from day one
- No public signup route — all user creation is invite-only via Clerk invitations
- AuditLog: write via Inngest background job, never in the critical path of a user-facing mutation

## RLS Policy SQL (apply as manual migration)

```sql
-- Enable RLS on all tenant tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE classes ENABLE ROW LEVEL SECURITY;
ALTER TABLE terms ENABLE ROW LEVEL SECURITY;
ALTER TABLE onboarding_states ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

-- Tenant isolation policy
CREATE POLICY tenant_isolation ON users
  USING (school_id = current_setting('app.current_school_id', true)::uuid);

CREATE POLICY tenant_isolation ON classes
  USING (school_id = current_setting('app.current_school_id', true)::uuid);

CREATE POLICY tenant_isolation ON terms
  USING (school_id = current_setting('app.current_school_id', true)::uuid);

CREATE POLICY tenant_isolation ON onboarding_states
  USING (school_id = current_setting('app.current_school_id', true)::uuid);

-- Audit logs: only the owning school can read; inserts allowed from service role
CREATE POLICY tenant_isolation ON audit_logs
  USING (school_id IS NULL OR school_id = current_setting('app.current_school_id', true)::uuid);
```

---

## Definition of Done

All criteria in `acceptance.md` pass.  
No unresolved `// ARCH-QUESTION:` comments.  
`DECISIONS.md` updated with any new decisions made during implementation.  
`STATE.md` updated: Sprint 002 marked complete.  
`CONTEXT.md` Sprint History updated.  
README updated with: setup instructions, env var list, how to run migrations, how to run tests.
