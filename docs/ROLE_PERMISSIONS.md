# ROLE_PERMISSIONS.md — EduManage Complete Permission Matrix

**Last updated:** 2026-05-29  
**Enforcement:** tRPC `protectedProcedure(allowedRoles[])` + Next.js middleware route guards + Postgres RLS

---

## How Permissions Are Enforced

Three layers — all must pass:

1. **Middleware** (`middleware.ts`) — route-level: blocks wrong role from reaching a page
2. **tRPC** (`protectedProcedure`) — procedure-level: throws FORBIDDEN if role not in allowedRoles
3. **RLS** (`school_id` policy) — data-level: wrong-tenant data never returned even if layers 1+2 bypassed

---

## Role Hierarchy Overview

```
SUPER_ADMIN          ← Platform operator (EduManage staff)
  └── HEADTEACHER    ← School owner / principal
        ├── DEAN_ACADEMICS
        ├── ACCOUNTANT
        ├── HR_MANAGER
        ├── IT_SUPPORT
        ├── PROCUREMENT_OFFICER
        ├── TRANSPORT_MANAGER
        ├── LIBRARIAN
        ├── SCHOOL_NURSE
        ├── CATERESS
        ├── MATRON
        ├── CLASS_TEACHER
        ├── SUBJECT_TEACHER
        ├── PARENT          ← External (own children only)
        └── STUDENT         ← External (own records only)
```

---

## Permission Matrix by Module

### School Administration

| Action | SA | HT | DEAN | IT | HR | Others |
|---|---|---|---|---|---|---|
| Create school | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| View all schools | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Edit school profile | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Invite/create users | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Deactivate users | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Change user role | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| View all users | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| View audit log | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Manage subscription | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| View storage usage | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ |

**SA** = Super Admin, **HT** = Headteacher

---

### Academic — Timetable

| Action | HT | DEAN | CT | ST | STUDENT | PARENT |
|---|---|---|---|---|---|---|
| Create timetable | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Edit timetable | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Publish timetable | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| View own class timetable | ✅ | ✅ | ✅ | ✅ | ✅ | ✅(child) |
| Assign substitute | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |

**CT** = Class Teacher, **ST** = Subject Teacher

---

### Academic — Attendance

| Action | HT | DEAN | CT | ST | STUDENT | PARENT |
|---|---|---|---|---|---|---|
| Mark attendance | ❌ | ❌ | ✅(own class) | ❌ | ❌ | ❌ |
| Edit attendance (<24h) | ❌ | ❌ | ✅(own class) | ❌ | ❌ | ❌ |
| Edit attendance (>7d override) | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| View class attendance | ✅ | ✅ | ✅(own) | ❌ | ❌ | ❌ |
| View own attendance | ❌ | ❌ | ❌ | ❌ | ✅ | ✅(child) |
| School-wide report | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |

IT Support can override 7-day lockout via `it.overrideAttendanceLock`.

---

### Academic — Assignments & Quizzes

| Action | HT | DEAN | CT | ST | STUDENT | PARENT |
|---|---|---|---|---|---|---|
| Create assignment | ❌ | ✅(view) | ✅ | ✅ | ❌ | ❌ |
| Publish assignment | ❌ | ❌ | ✅(own) | ✅(own) | ❌ | ❌ |
| Submit assignment | ❌ | ❌ | ❌ | ❌ | ✅(own class) | ❌ |
| Mark assignment | ❌ | ❌ | ✅(own) | ✅(own) | ❌ | ❌ |
| View own submission | ❌ | ❌ | ❌ | ❌ | ✅ | ✅(child, after marking) |
| Create quiz | ❌ | ✅(view) | ✅ | ✅ | ❌ | ❌ |
| Attempt quiz | ❌ | ❌ | ❌ | ❌ | ✅(own class, within window) | ❌ |
| Grade short answers | ❌ | ❌ | ✅(own quiz) | ✅(own quiz) | ❌ | ❌ |
| Release quiz results | ❌ | ❌ | ✅(own quiz) | ✅(own quiz) | ❌ | ❌ |
| View quiz results | ✅ | ✅ | ✅(own quiz) | ✅(own quiz) | ✅(own, after release) | ✅(child, after release) |

---

### Academic — Exams

| Action | HT | DEAN | CT | ST | STUDENT | PARENT |
|---|---|---|---|---|---|---|
| Create exam | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| Upload exam paper | ❌ | ❌ | ✅(own exam) | ✅(own exam) | ❌ | ❌ |
| View exam paper (pre-release) | ✅ | ✅ | ✅(own) | ✅(own) | ❌ | ❌ |
| View exam paper (post-release) | ✅ | ✅ | ✅ | ✅ | ✅(own class) | ❌ |
| Enter results | ❌ | ❌ | ✅(own) | ✅(own) | ❌ | ❌ |
| Approve results | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Release results | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| View results (after release) | ✅ | ✅ | ✅ | ✅ | ✅(own) | ✅(child) |

---

### Analytics

| View | HT | DEAN | ACCT | CT | ST | STUDENT | PARENT |
|---|---|---|---|---|---|---|---|
| School-wide analytics | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Class analytics | ✅ | ✅ | ❌ | ✅(own class) | ✅(own subject) | ❌ | ❌ |
| Student analytics | ✅ | ✅ | ❌ | ✅(own class) | ✅(own subject) | ✅(own) | ✅(child) |
| Financial analytics | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| AI insights | ✅ | ✅ | ✅(financial) | ✅(own class) | ❌ | ❌ | ❌ |

---

### Calendar

| Action | ALL ROLES |
|---|---|
| View personal calendar | ✅ |
| Create personal event | ✅ |
| Create class event | CT only |
| Create school-wide event | HT + DEAN only |
| Create club event | Club patron only |
| Export .ics | ✅ (own events only) |
| View others' personal events | ❌ (never) |

---

### Financial

| Action | HT | DEAN | ACCT | PROC | PARENT | STUDENT |
|---|---|---|---|---|---|---|
| Configure fee structure | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Approve fee structure | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| View own fee statement | ❌ | ❌ | ✅(all) | ❌ | ✅(child) | ✅(own) |
| Initiate Mpesa payment | ❌ | ❌ | ✅(manual) | ❌ | ✅(own child) | ❌ |
| Record manual payment | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Download Mpesa statement | ❌ | ❌ | ✅(all) | ❌ | ✅(child only) | ❌ |
| View all payments | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Create salary run | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Approve salary run | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Generate KRA reports | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Approve budget requests | ✅ | ❌ | ✅(budget check) | ❌ | ❌ | ❌ |
| Upload bank statement | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| View subscription status | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

---

### Procurement

| Action | HT | DEAN | ACCT | PROC | ANY STAFF |
|---|---|---|---|---|---|
| Raise request (own dept) | ✅ | ✅ | ✅ | ✅ | ✅ |
| Approve as HOD | HOD role | HOD role | HOD role | HOD role | HOD role |
| Approve as Procurement Officer | ❌ | ❌ | ❌ | ✅ | ❌ |
| Budget-check approval | ❌ | ❌ | ✅ | ❌ | ❌ |
| Final approval (>threshold) | ✅ | ❌ | ❌ | ❌ | ❌ |
| Place order | ❌ | ❌ | ❌ | ✅ | ❌ |
| Confirm delivery | ❌ | ❌ | ❌ | ✅ | ❌ |
| View all requests | ✅ | ✅ | ✅ | ✅ | own only |
| Manage inventory | ❌ | ❌ | ❌ | ✅ | ❌ |

---

### Communications

| Action | ALL STAFF | STUDENT | PARENT |
|---|---|---|---|
| Send direct message | ✅ | ✅(to teachers only) | ✅(to school staff) |
| Create group | HT + DEAN + CT only | ❌ | ❌ |
| Send to group >10 | ✅ (with safety gate) | ❌ | ❌ |
| Broadcast to all users | HT only | ❌ | ❌ |
| View notification preferences | ✅ | ✅ | ✅ |

---

### Library

| Action | HT | DEAN | LIB | CT/ST | STUDENT | PARENT |
|---|---|---|---|---|---|---|
| Upload school resource | ❌ | ❌ | ✅(direct) | ✅(for review) | ❌ | ❌ |
| Approve/reject resource | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Search school library | ✅ | ✅ | ✅ | ✅ | ✅(own class) | ❌ |
| Upload personal item | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| View child's library activity | ❌ | ❌ | ❌ | ❌ | ❌ | ✅(child) |
| Suggest resource to child | ❌ | ❌ | ❌ | ❌ | ❌ | ✅(child) |
| View storage usage | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |

---

### Nursing (Confidential)

| Action | HT | NURSE | CT/ST | PARENT | STUDENT |
|---|---|---|---|---|---|
| Log health visit | ❌ | ✅ | ❌ | ❌ | ❌ |
| View health visits | ✅ | ✅ | ❌ | ❌ | ❌ |
| Flag follow-up | ❌ | ✅ | ❌ | ❌ | ❌ |
| Receive health alert | ✅ | ❌ | ✅(class teacher) | ✅(own child) | ❌ |
| Manage medical inventory | ❌ | ✅ | ❌ | ❌ | ❌ |

---

### HR

| Action | HT | HR | STAFF (own) |
|---|---|---|---|
| Create staff record | ❌ | ✅ | ❌ |
| View all staff records | ✅ | ✅ | own only |
| Upload staff documents | ❌ | ✅ | ❌ |
| View staff documents | ✅ | ✅ | ❌ |
| Initiate appraisal | ❌ | ✅ | ❌ |
| Submit self-appraisal | ❌ | ❌ | ✅ |
| Review appraisal (line mgr) | ❌ | ❌ | ✅(if line manager) |
| Approve appraisal | ✅ | ❌ | ❌ |
| Update onboarding checklist | ❌ | ✅ | ❌ |
| Confirm payroll headcount | ❌ | ✅ | ❌ |

---

### AI Features (Premium Tier Only)

| Feature | HT | DEAN | ACCT | CT | ST |
|---|---|---|---|---|---|
| Generate term reports | ✅ | ✅ | ❌ | ❌ | ❌ |
| View AI insights | ✅ | ✅ | ✅(financial) | ✅(own class) | ✅(own subject) |
| Approve + publish AI report | ✅ | ❌ | ❌ | ❌ | ❌ |
| Suggest assignment feedback | ❌ | ❌ | ❌ | ✅(own) | ✅(own) |
| AI timetable review | ❌ | ✅ | ❌ | ❌ | ❌ |
| Lesson plan assistant | ❌ | ❌ | ❌ | ✅ | ✅ |
| Financial AI narrative | ❌ | ❌ | ✅ | ❌ | ❌ |
| View AI usage dashboard | ✅ | ❌ | ❌ | ❌ | ❌ |

---

## tRPC `protectedProcedure` Implementation Reference

```typescript
// server/trpc.ts
const ROLE_GROUPS = {
  ALL_STAFF: ['HEADTEACHER','DEAN_ACADEMICS','CLASS_TEACHER','SUBJECT_TEACHER',
              'ACCOUNTANT','PROCUREMENT_OFFICER','IT_SUPPORT','HR_MANAGER',
              'TRANSPORT_MANAGER','LIBRARIAN','SCHOOL_NURSE','CATERESS','MATRON'],
  ALL_USERS: [...ALL_STAFF, 'PARENT', 'STUDENT'],
  ACADEMIC_STAFF: ['HEADTEACHER','DEAN_ACADEMICS','CLASS_TEACHER','SUBJECT_TEACHER'],
  FINANCE_STAFF: ['HEADTEACHER','ACCOUNTANT'],
  APPROVERS: ['HEADTEACHER','DEAN_ACADEMICS','ACCOUNTANT'],
  ADMIN: ['HEADTEACHER','IT_SUPPORT'],
} as const

export const protectedProcedure = (allowedRoles: UserRole[]) =>
  t.procedure.use(async ({ ctx, next }) => {
    if (!ctx.userId) throw new TRPCError({ code: 'UNAUTHORIZED' })
    if (!allowedRoles.includes(ctx.role)) throw new TRPCError({ code: 'FORBIDDEN' })
    return next()
  })

// Premium-tier gate (additional to role check)
export const premiumProcedure = (allowedRoles: UserRole[]) =>
  protectedProcedure(allowedRoles).use(async ({ ctx, next }) => {
    const sub = await ctx.prisma.schoolSubscription.findUnique({
      where: { schoolId: ctx.schoolId }
    })
    if (sub?.tier !== 'PREMIUM') {
      throw new TRPCError({ code: 'FORBIDDEN', message: 'Requires Premium subscription' })
    }
    return next()
  })
```

---

## Middleware Route Guard Map

```typescript
// middleware.ts route protection
const ROUTE_ROLES: Record<string, UserRole[]> = {
  '/admin':           ['SUPER_ADMIN'],
  '/financial':       ['HEADTEACHER','ACCOUNTANT'],
  '/academic/exams':  ['HEADTEACHER','DEAN_ACADEMICS','CLASS_TEACHER','SUBJECT_TEACHER'],
  '/analytics/school':['HEADTEACHER','DEAN_ACADEMICS'],
  '/procurement':     ['HEADTEACHER','ACCOUNTANT','PROCUREMENT_OFFICER',...ALL_DEPT_HEADS],
  '/hr':              ['HEADTEACHER','HR_MANAGER'],
  '/nursing':         ['HEADTEACHER','SCHOOL_NURSE'],
  '/boarding':        ['HEADTEACHER','MATRON'],
  '/transport':       ['HEADTEACHER','TRANSPORT_MANAGER'],
  '/kitchen':         ['HEADTEACHER','CATERESS'],
  '/library/admin':   ['HEADTEACHER','LIBRARIAN','IT_SUPPORT'],
}
// Student + Parent routes: only their own data (enforced by RLS + tRPC ownership check)
```
