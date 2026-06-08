# Blueprint — Sprint 002: Auth & Multi-Tenancy

**Project:** EduManage  
**Sprint:** 002  
**Status:** Approved  
**Last updated:** 2026-05-29

---

## Sprint Scope Summary

Build the complete authentication and tenancy foundation: Clerk organisation-per-school, all 16 role definitions, RLS policies on the initial schema, unique ID generation, school onboarding wizard, and role-based dashboard routing. Nothing else is built until this sprint passes all acceptance criteria.

---

## Technology Stack (this sprint)

| Concern | Choice | Justification |
|---|---|---|
| Auth | Clerk @clerk/nextjs | Org-per-school tenancy model |
| DB | Neon + Prisma 5 | Schema with RLS, initial migrations |
| API | tRPC v11 | `protectedProcedure` with role checking |
| Middleware | Next.js middleware | JWT validation, school_id injection |
| UI | shadcn/ui + Tailwind | Onboarding wizard, dashboard shells |
| Validation | Zod | All tRPC inputs |
| File upload | AWS S3 presigned POST | School logo upload |

---

## Domain Model (this sprint)

### School

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | |
| clerkOrgId | String | UNIQUE NOT NULL | Maps to Clerk org |
| name | String | NOT NULL | |
| slug | String | UNIQUE NOT NULL | URL-safe |
| emailDomain | String? | | e.g. "stmarys.ac.ke" |
| curriculumType | Enum | NOT NULL | CBC \| IGCSE \| BOTH |
| subscriptionTier | Enum | NOT NULL | BASIC \| STANDARD \| PREMIUM |
| subscriptionStatus | Enum | NOT NULL | TRIAL \| ACTIVE \| SUSPENDED |
| logoUrl | String? | | S3 URL |
| county | String | NOT NULL | |
| onboardingComplete | Boolean | default false | |
| createdAt | DateTime | default now() | |

### User

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | |
| clerkUserId | String | UNIQUE NOT NULL | |
| uniqueId | String | UNIQUE NOT NULL | EDU-XXX-YYY-00000 |
| schoolId | UUID | FK → School | NULL for SUPER_ADMIN |
| email | String | NOT NULL | |
| firstName | String | NOT NULL | |
| lastName | String | NOT NULL | |
| role | UserRole | NOT NULL | Enum, all 16 roles |
| isActive | Boolean | default true | |
| createdAt | DateTime | default now() | |

### OnboardingState

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | |
| schoolId | UUID | FK → School UNIQUE | |
| currentStep | Int | default 1 | 1–5 |
| completedSteps | Int[] | | |
| stepData | Json | | Persisted wizard data |
| updatedAt | DateTime | | |

### AuditLog

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | |
| schoolId | UUID | nullable (platform events) | |
| actorId | UUID | NOT NULL | User.id |
| action | String | NOT NULL | e.g. "user.created" |
| entityType | String | NOT NULL | |
| entityId | UUID | NOT NULL | |
| metadata | Json? | | Non-PII context |
| createdAt | DateTime | default now() | |

### Relationships

```
School 1──────< User
School 1──────1 OnboardingState
User   1──────< AuditLog (as actor)
```

---

## API Contracts (this sprint)

### Create School (Super Admin only)

**Method:** `school.create` (tRPC mutation)  
**Auth:** SUPER_ADMIN role required

**Input:**
```json
{
  "name": "string",
  "curriculumType": "CBC | IGCSE | BOTH",
  "county": "string",
  "headteacherEmail": "string",
  "headteacherFirstName": "string",
  "headteacherLastName": "string"
}
```

**Output:**
```json
{
  "schoolId": "uuid",
  "slug": "string",
  "clerkOrgId": "string",
  "headteacherInviteSent": true
}
```

**Errors:**
| Code | Condition |
|---|---|
| 409 | School name or slug already exists |
| 500 | Clerk org creation failed — DB rollback |

---

### Complete Onboarding Step

**Method:** `school.onboarding.saveStep` (tRPC mutation)  
**Auth:** HEADTEACHER role, own school

**Input:**
```json
{
  "step": 1,
  "data": { "...step-specific fields..." }
}
```

**Output:**
```json
{
  "nextStep": 2,
  "isComplete": false
}
```

---

### Invite Staff Member

**Method:** `school.inviteUser` (tRPC mutation)  
**Auth:** HEADTEACHER | IT_SUPPORT

**Input:**
```json
{
  "email": "string",
  "role": "UserRole",
  "firstName": "string",
  "lastName": "string"
}
```

**Output:**
```json
{
  "userId": "uuid",
  "uniqueId": "string",
  "inviteSent": true
}
```

---

## Cross-Cutting Concerns

**Auth:** All tRPC procedures use `protectedProcedure(allowedRoles: UserRole[])` helper. Non-authenticated requests return 401. Wrong role returns 403.

**Validation:** Zod schemas on all inputs. Errors return structured `{ field, message }` array.

**Error handling:** tRPC error codes (BAD_REQUEST, UNAUTHORIZED, FORBIDDEN, INTERNAL_SERVER_ERROR). No stack traces in production responses.

**Logging:** Structured JSON logs. No PII. AuditLog written for: school.created, user.created, user.deactivated, role.changed, onboarding.completed.

**RLS:** `prismaWithRLS(schoolId)` wrapper function sets `SET LOCAL app.current_school_id = '...'` on every transaction.

---

## Folder Structure (Sprint 002 additions)

```
app/
├── (auth)/
│   ├── sign-in/page.tsx
│   └── sign-up/page.tsx  (invite-only — no public signup)
├── (platform)/
│   └── admin/
│       ├── schools/page.tsx        ← Super Admin school list
│       └── schools/new/page.tsx    ← Create school form
├── (school)/
│   └── [schoolSlug]/
│       ├── onboarding/page.tsx     ← 5-step wizard
│       └── dashboard/page.tsx     ← Role-aware dashboard shell
server/
├── routers/
│   └── school.ts                  ← create, inviteUser, onboarding.*
├── context.ts                     ← { schoolId, userId, role, prisma }
├── trpc.ts                        ← protectedProcedure helper
└── db.ts                          ← prismaWithRLS wrapper
prisma/
└── migrations/
    └── 0001_init_auth_tenancy/
```

---

## Architectural Decisions (this sprint)

| Decision | Choice | Reason |
|---|---|---|
| Clerk org = school tenant | 1:1 mapping | orgId used as schoolId everywhere |
| No public signup | Invite-only for staff | Schools control user access |
| Unique ID format | EDU-[SCHOOL]-[ROLE]-[SEQ] | Human-readable, role-evident |
| RLS set via SET LOCAL | Per-transaction | Safest; no connection pool bleed |
| Wizard state in DB | OnboardingState table | Resumable across sessions |

---

## Risks (this sprint)

- Clerk webhook for org events must be configured before testing school creation
- RLS `SET LOCAL` must be tested under connection pooling (PgBouncer/Neon pooler) — pooled connections can leak settings if not using transaction mode
