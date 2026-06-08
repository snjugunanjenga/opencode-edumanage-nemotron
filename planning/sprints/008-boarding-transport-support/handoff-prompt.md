# Builder Handoff — Sprint 008: Boarding, Transport, Nursing, Kitchen & HR

Paste everything below into Claude Code. Sprint 007 must be fully passing.

---

You are building Sprint 008 of EduManage — the final sprint. After this, all 16 roles are fully operational and the platform is complete.

## Pre-read (mandatory, in order):
1. `AGENTS.md` + `CONTEXT.md`
2. `planning/DOMAIN.md` — re-read the 5 new department roles
3. `planning/DECISIONS.md` — all decisions, especially DEC-001 (RLS)
4. `planning/RISKS.md` — RISK-008 (health visit confidentiality)
5. `planning/sprints/008-boarding-transport-support/requirements.md`
6. `planning/sprints/008-boarding-transport-support/blueprint.md`
7. `planning/sprints/008-boarding-transport-support/acceptance.md`
8. `docs/ROLE_PERMISSIONS.md` — nursing and HR permission tables

## Agent routing:
- Prisma models, tRPC routers → `nextjs-expert`
- Dorm map UI, meal planner grid, trip timeline → `react-frontend`
- Role-based confidentiality guards → `clerk-auth`
- Inngest jobs, RLS SQL, partial index → `devops`

## Summarise before coding — output:
1. All new files (full paths)
2. All modified files + what changes
3. Prisma migration name
4. Manual SQL migration file name + contents
5. Inngest job names
6. The `prismaWithRLS` signature change and all call sites that need updating
7. Any `// ARCH-QUESTION:` items

**Wait for operator approval. Then build.**

---

## Critical implementation order

### Step 1: Update prismaWithRLS FIRST (breaking change)
The `prismaWithRLS` function must accept `role` as a third parameter and set `app.current_role` in every DB session. This change affects `server/context.ts` and EVERY tRPC context instantiation. Do this before any Sprint 008 code — otherwise health visit confidentiality cannot work.

```typescript
// server/db.ts
export function prismaWithRLS(schoolId: string, userId: string, role: string) {
  return prisma.$extends({
    query: {
      $allModels: {
        async $allOperations({ args, query }) {
          return prisma.$transaction(async (tx) => {
            await tx.$executeRaw`
              SELECT 
                set_config('app.current_school_id', ${schoolId}, true),
                set_config('app.current_user_id', ${userId}, true),
                set_config('app.current_role', ${role}, true)
            `
            return query(args)
          })
        },
      },
    },
  })
}
```

Update `server/context.ts` to extract `role` from Clerk JWT and pass to `prismaWithRLS`.

### Step 2: Prisma migration
```
prisma migrate dev --name sprint_008_departments
```

### Step 3: Manual SQL migration
File: `prisma/migrations/manual/007_rls_sprint008.sql`
- Standard RLS for all new tables EXCEPT health_visits and staff_documents
- `health_visits`: role-based policy (`SCHOOL_NURSE` or `HEADTEACHER` only)
- `staff_documents`: role-based policy (`HR_MANAGER` or `HEADTEACHER` only)
- Partial unique index: `dorm_active_bed_unique`

### Step 4: tRPC routers
In this order:
1. `server/routers/boarding.ts`
2. `server/routers/transport.ts`
3. `server/routers/nursing.ts` — MUST use `protectedProcedure(['SCHOOL_NURSE', 'HEADTEACHER'])` on ALL read procedures
4. `server/routers/kitchen.ts`
5. `server/routers/hr.ts` — `hr.getDocuments` MUST use `protectedProcedure(['HR_MANAGER', 'HEADTEACHER'])`
6. Update `server/routers/index.ts`

### Step 5: Inngest jobs (5 new jobs)
- `vehicle-insurance-expiry` (weekly cron)
- `contract-expiry-alert` (monthly cron)
- `kitchen-reorder-check` (daily cron + idempotency)
- `appraisal-cycle-start` (manual trigger)
- `trip-approved-calendar` (trip approved event)

### Step 6: Pages and components
All 5 department pages + shared components.

### Step 7: Integration tests (write BEFORE marking done)
- Nursing confidentiality: CLASS_TEACHER gets FORBIDDEN at tRPC layer
- Nursing confidentiality: CLASS_TEACHER gets 0 rows at RLS layer (direct DB test)
- Dorm double-booking: concurrent insert test — DB rejects second insert
- HR documents: CLASS_TEACHER gets FORBIDDEN on getDocuments
- Full regression suite

---

## Critical constraints

1. **Health visit confidentiality is non-negotiable** — enforce at BOTH tRPC (role check) AND Postgres RLS (role policy). Test both layers explicitly. This is a patient safety concern.

2. **Staff documents** — HR + HT only — strip `s3Key` from all non-HR/HT responses. Never generate presigned URL for other roles.

3. **Dorm double-booking** — rely on DB partial unique index as the final guard. Application-level check is a UX convenience only — the DB is the authority.

4. **Kitchen auto-procurement** — DRAFT status ONLY. Inngest creates the request; Cateress reviews and submits manually. Never auto-submit procurement.

5. **Trip calendar events** — only created AFTER full approval. Draft/pending trips have no calendar events.

6. **Appraisal self-submission** — validate `appraisal.userId === ctx.userId` before allowing submission. Never allow submitting on behalf of another.

7. **prismaWithRLS breaking change** — update ALL call sites. Search codebase for `prismaWithRLS(` and add `role` param everywhere. Missing updates = wrong role context = RLS policies misfire.

8. **Kitchen reorder idempotency** — check for existing DRAFT/PENDING request for same item in same week before creating new one. No duplicate procurement drafts.

---

## Final platform verification

After all tests pass, verify the complete platform with a smoke test covering all 16 roles:

```
SUPER_ADMIN    → creates school, views all schools
HEADTEACHER    → onboarding wizard, dashboard, salary approval, AI reports
DEAN           → timetable, exam approval, leave queue, AI review
CLASS_TEACHER  → attendance, assignments, quizzes, calendar
SUBJECT_TEACHER → lesson plans, exams, library resources
ACCOUNTANT     → fee structure, Mpesa payment, payroll, KRA reports
PROCUREMENT    → create order, inventory, receive delivery
IT_SUPPORT     → storage dashboard, user management
HR_MANAGER     → staff records, documents, appraisals
TRANSPORT_MANAGER → vehicle list, trip plan, budget
LIBRARIAN      → resource review queue, storage usage
SCHOOL_NURSE   → health visit log, medical inventory
CATERESS       → meal plan, kitchen inventory, budget
MATRON         → dorm map, bed allocation, cleaning roster
PARENT         → fee statement, pay Mpesa, child analytics
STUDENT        → assignments, quizzes, results, library, calendar
```

---

## Definition of Done

All AC-001 through AC-006 pass.  
Health visit confidentiality tested at both tRPC AND RLS layers.  
Dorm double-booking tested with concurrent DB insert attempt.  
prismaWithRLS updated at ALL call sites — no missed updates.  
Full regression suite passing (all 8 sprints).  
16-role smoke test passing.  
`CONTEXT.md` Sprint History: all 8 sprints marked ✅ complete.  
`STATE.md` updated: "Platform complete — ready for first school onboarding."
