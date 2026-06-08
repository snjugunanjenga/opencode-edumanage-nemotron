# CONTEXT.md — EduManage

**Last updated:** 2026-05-29 — Session 3 (Final)
**Current phase:** All 8 sprints planned. Builder can begin Sprint 002.
**Stack:** Locked. See DECISIONS.md.

---

## What We Are Building

EduManage is a multi-tenant SaaS school management platform for Kenyan private schools (CBC + IGCSE). It replaces every disconnected tool a school uses — WhatsApp, Excel, paper registers, manual fee tracking — with one unified platform covering academic management, financial operations, procurement, HR, boarding, transport, library, nursing, kitchen, and AI-driven automation.

**Business model:** Term-based SaaS subscriptions (Basic Ksh 25K / Standard Ksh 50K / Premium Ksh 100K per term).  
**Target market:** 40,000+ formal schools in Kenya. Year 1 goal: 15 schools at Standard tier.  
**First revenue:** After Sprint 004 ships.

---

## Technology Stack (Locked — DEC-001 through DEC-017)

| Concern | Choice |
|---|---|
| Framework | Next.js 16 App Router + TypeScript strict |
| Internal API | tRPC v11 |
| Database | Neon (serverless Postgres) + Prisma 5 |
| Multi-tenancy | Row-Level Security, `school_id` on every table (DEC-001) |
| Auth | Clerk — one Organisation per school, orgId = schoolId (DEC-002) |
| Async jobs | Inngest |
| File storage | AWS S3 eu-west-1 (DEC-013) |
| AI | Google Gemini 1.5 Pro — Premium tier only (DEC-005) |
| Payments | Daraja Mpesa + Stripe (DEC-006) |
| Email | Resend |
| Hosting | Vercel |
| Charts | Recharts |
| Calendar UI | react-big-calendar + date-fns |
| Styling | Tailwind CSS + shadcn/ui |

---

## 16 User Roles

```
Platform: SUPER_ADMIN
School Admin: HEADTEACHER, IT_SUPPORT, HR_MANAGER
Academic: DEAN_ACADEMICS, CLASS_TEACHER, SUBJECT_TEACHER
Financial: ACCOUNTANT
Operations: PROCUREMENT_OFFICER, TRANSPORT_MANAGER, LIBRARIAN,
            SCHOOL_NURSE, CATERESS, MATRON
External: PARENT, STUDENT
```

---

## Non-Negotiable Architectural Rules

1. Every table: `school_id UUID NOT NULL` + RLS tenant isolation policy
2. `school_id = Clerk orgId` — no mapping table
3. All DB writes via `prismaWithRLS(schoolId)` wrapper — never raw `prisma.xxx`
4. All file uploads: presigned S3 POST only — never stream through Next.js
5. All Gemini calls: via Inngest jobs only — never synchronous in tRPC
6. Premium gate: `premiumProcedure` middleware on every AI endpoint
7. AI banner: `<AIBanner />` on all AI-generated content — no exceptions
8. Exam papers: encrypted at rest, time-locked access (DEC-011)
9. Health visits: SCHOOL_NURSE + HEADTEACHER access only — dual RLS + role check
10. Financial payments: immutable after `status = COMPLETED`

---

## Sprint History

| Sprint | Name | Status | Key Entities |
|---|---|---|---|
| 001 | Discovery & Architecture | ✅ Planning complete | All planning docs |
| 002 | Auth & Multi-Tenancy | 📋 Ready | School, User, OnboardingState, AuditLog |
| 003 | Academic Core | 📋 Ready | Subject, Timetable, Attendance, Assignment, Quiz, Exam, Analytics, Calendar, LessonPlan, Syllabus |
| 004 | Financial & Mpesa | 📋 Ready | FeeStructure, FeePayment, SalaryRun, TaxRecord, BankStatement, SchoolSubscription |
| 005 | Communications & Procurement | 📋 Ready | MessageThread, Notification, Club, LeaveRequest, ProcurementRequest, InventoryItem |
| 006 | Library & Resources | 📋 Ready | LibraryResource, StudentLibraryItem, StorageUsage |
| 007 | AI Automation | 📋 Ready | AIReport, AIInsight, AIUsageLog |
| 008 | Boarding/Transport/Nursing/Kitchen/HR | 📋 Ready | Dorm, Vehicle, Trip, HealthVisit, MealPlan, KitchenInventory, StaffRecord, PerformanceAppraisal |

**Total entities across all sprints: ~75 Prisma models**

---

## Builder Agents Available

| Agent | File | Use When |
|---|---|---|
| Software Architect | `.claude/agents/software-architect.md` | Planning, sprint scoping, decisions, new features |
| Next.js Expert | `.claude/agents/nextjs-expert.md` | Pages, API routes, server components, tRPC routers |
| React Frontend | `.claude/agents/react-frontend.md` | Client components, charts, calendar, forms, quiz UI |
| Clerk Auth | `.claude/agents/clerk-auth.md` | Auth flows, middleware, role guards, invitations |
| DevOps | `.claude/agents/devops.md` | CI/CD, S3, Inngest, monitoring, GitHub Actions |

---

## Key Business Rules (Summary)

- All staff + students use school-assigned email addresses
- Every user gets a `unique_id` at registration: `EDU-[SCHOOL]-[ROLE]-[NNNNN]`
- CBC + IGCSE curriculum both supported (configurable per school)
- KRA tax reports exported for manual iTax filing (no direct API, DEC-009)
- Salary disbursed via school's bank; EduManage generates payslips only (DEC-014)
- Storage quotas: Basic 50GB / Standard 200GB / Premium 1TB (DEC-010)
- Procurement requires full multi-level approval chain; all steps immutable
- Exam papers: password-required upload, Dean sets release timestamp
- Health visits: strictly confidential — only Nurse + Headteacher can read
- Dorm bed: partial unique index prevents double-allocation at DB level

---

## Open Questions

| Q | Question | Default | Needed By |
|---|---|---|---|
| Q-007 | Quiz question types | MCQ + T/F + Short Answer | Sprint 003 (defaulted) |
| Q-008 | Analytics data retention | 2yr raw → monthly rollups | Sprint 003 (defaulted) |

---

## What Comes Next

**Builder:** Paste `planning/sprints/002-auth-multitenancy/handoff-prompt.md` into Claude Code.  
**After Sprint 002 ACs pass:** Paste Sprint 003 handoff-prompt.md.  
**First revenue gate:** Sprint 004 complete + 15 pilot schools onboarded.  
**Full platform:** Sprint 008 complete.
