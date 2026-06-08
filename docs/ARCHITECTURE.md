# ARCHITECTURE.md — EduManage System Architecture

**Version:** 1.0  
**Date:** 2026-05-29

---

## Overview

EduManage is a Next.js 14 fullstack multi-tenant SaaS application. The architecture follows **Feature-Sliced Clean Architecture** — domain logic is isolated from infrastructure, features are self-contained modules, and the multi-tenancy boundary is enforced at the database layer via PostgreSQL Row-Level Security.

---

## System Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│  Next.js 14 App Router (RSC + Client Components)                │
│  Tailwind CSS  |  shadcn/ui  |  Recharts  |  React Hook Form    │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS
┌──────────────────────────▼──────────────────────────────────────┐
│                      VERCEL EDGE                                 │
│  Middleware: Clerk auth check + school_id injection              │
│  Edge Config: feature flags per subscription tier                │
└──────────────┬───────────────────────────┬──────────────────────┘
               │                           │
┌──────────────▼──────────┐   ┌────────────▼──────────────────────┐
│   tRPC API LAYER         │   │   WEBHOOK HANDLERS (API Routes)   │
│   /server/routers/       │   │   /api/webhooks/mpesa             │
│   school | academic      │   │   /api/webhooks/stripe            │
│   financial | comms      │   │   /api/webhooks/clerk             │
│   procurement | library  │   └────────────┬──────────────────────┘
│   boarding | hr | ai     │                │
└──────────────┬──────────┘                │
               │                           │
┌──────────────▼───────────────────────────▼──────────────────────┐
│                    INNGEST (Async Workers)                       │
│  Fee reminder jobs  |  Report generation  |  AI analysis        │
│  Notification delivery  |  Mpesa retry queue                    │
│  Timetable conflict detection  |  Approval escalation           │
└──────────────┬──────────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────────┐
│                    DATA + SERVICES LAYER                        │
│                                                                 │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │  Neon Postgres   │  │   AWS S3     │  │  Google Gemini API │ │
│  │  (Prisma ORM)    │  │  af-south-1  │  │  1.5 Pro           │ │
│  │  RLS enforced    │  │  Presigned   │  │  Reports + Analysis│ │
│  └─────────────────┘  │  URLs        │  └────────────────────┘ │
│                        └──────────────┘                         │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │  Clerk Auth      │  │  Daraja      │  │  Stripe            │ │
│  │  Orgs + Roles    │  │  Mpesa API   │  │  Subscriptions     │ │
│  │  JWT + Metadata  │  │  STK + C2B   │  │  Webhooks          │ │
│  └─────────────────┘  └──────────────┘  └────────────────────┘ │
│                                                                 │
│  ┌─────────────────┐                                            │
│  │  Resend          │                                            │
│  │  Transactional   │                                            │
│  │  Email           │                                            │
│  └─────────────────┘                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Multi-Tenancy Architecture

### Tenancy Model: Row-Level Security (RLS)

Every table in the database includes `school_id UUID NOT NULL`. PostgreSQL RLS policies enforce that all queries are scoped to the authenticated user's school.

```sql
-- Example RLS policy (applied to every table)
CREATE POLICY "tenant_isolation" ON students
  USING (school_id = current_setting('app.current_school_id')::uuid);
```

### Clerk → school_id Flow

```
1. User logs in via Clerk
2. Clerk JWT includes: { userId, orgId (= school_id), role, unique_id }
3. Next.js middleware extracts orgId, sets app.current_school_id on DB connection
4. All Prisma queries automatically filtered by RLS
5. tRPC context injects { schoolId, userId, role } into every procedure
```

### School Onboarding Flow (Super Admin)

```
Super Admin creates school →
  Clerk Org created (orgId = schoolId) →
  schools table row inserted →
  Headteacher user created + invited →
  Headteacher completes setup wizard:
    → School profile (name, logo, type, curriculum)
    → Academic year + terms configured
    → Classes + streams created
    → Staff invited (email → Clerk invite → role assigned)
    → Students imported (CSV) or added manually
```

---

## Authentication & Authorisation

### Auth Flow

```
Browser → Clerk hosted login →
  JWT issued (contains: userId, orgId, publicMetadata.role, publicMetadata.unique_id) →
  Next.js middleware validates JWT →
  Middleware attaches { schoolId, role } to request →
  tRPC context receives school-scoped auth object →
  Procedure-level role checks enforced
```

### Role Permission Matrix (abbreviated)

| Permission | Super Admin | Headteacher | Dean | Class Teacher | Accountant | Procurement | Parent | Student |
|---|---|---|---|---|---|---|---|---|
| View all schools | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Manage school users | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Approve exam results | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| View own child records | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Submit assignments | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Approve budgets | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Place procurement orders | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |

Full matrix: see `docs/ROLE_PERMISSIONS.md` (Sprint 002 deliverable).

---

## Folder Structure

```
edumanage/
├── app/                          # Next.js App Router
│   ├── (auth)/                   # Clerk auth pages
│   ├── (platform)/               # Super Admin routes
│   ├── (school)/                 # All school-scoped routes
│   │   ├── [schoolSlug]/
│   │   │   ├── dashboard/
│   │   │   ├── academic/
│   │   │   │   ├── classes/
│   │   │   │   ├── attendance/
│   │   │   │   ├── exams/
│   │   │   │   ├── timetable/
│   │   │   │   └── assignments/
│   │   │   ├── financial/
│   │   │   │   ├── fees/
│   │   │   │   ├── payments/
│   │   │   │   ├── budgets/
│   │   │   │   ├── reports/
│   │   │   │   └── tax/
│   │   │   ├── procurement/
│   │   │   ├── library/
│   │   │   ├── communications/
│   │   │   ├── hr/
│   │   │   ├── boarding/
│   │   │   ├── transport/
│   │   │   ├── nursing/
│   │   │   ├── kitchen/
│   │   │   └── settings/
│   └── api/
│       ├── trpc/[trpc]/
│       └── webhooks/
│           ├── mpesa/
│           ├── stripe/
│           └── clerk/
├── server/
│   ├── routers/                  # tRPC routers (one per domain)
│   │   ├── school.ts
│   │   ├── academic.ts
│   │   ├── financial.ts
│   │   ├── procurement.ts
│   │   ├── library.ts
│   │   ├── communications.ts
│   │   ├── hr.ts
│   │   ├── boarding.ts
│   │   ├── transport.ts
│   │   ├── nursing.ts
│   │   ├── kitchen.ts
│   │   └── ai.ts
│   ├── context.ts                # tRPC context (auth + db)
│   ├── trpc.ts                   # tRPC init
│   └── db.ts                     # Prisma client (school-scoped)
├── domain/                       # Pure business logic (no framework deps)
│   ├── academic/
│   ├── financial/
│   ├── procurement/
│   └── shared/
├── inngest/                      # Async job definitions
│   ├── functions/
│   └── client.ts
├── lib/
│   ├── mpesa/                    # Daraja API client
│   ├── stripe/                   # Stripe client
│   ├── gemini/                   # Google AI client
│   ├── s3/                       # AWS S3 client + presigned URLs
│   ├── resend/                   # Email client
│   └── utils/
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── components/
│   ├── ui/                       # shadcn/ui components
│   ├── shared/                   # Cross-feature components
│   └── [feature]/                # Feature-specific components
├── planning/                     # This folder — architect artifacts
└── docs/                         # Technical docs
```

---

## Database Schema (Core Tables)

```prisma
// Core tenancy
model School {
  id              String   @id @default(uuid())
  clerkOrgId      String   @unique
  name            String
  slug            String   @unique
  curriculumType  CurriculumType  // CBC | IGCSE | BOTH
  subscriptionTier SubscriptionTier
  subscriptionStatus SubscriptionStatus
  logoUrl         String?
  county          String
  createdAt       DateTime @default(now())
  
  users           User[]
  classes         Class[]
  terms           Term[]
  subjects        Subject[]
  // ... all domain relations
}

model User {
  id          String   @id @default(uuid())
  clerkUserId String   @unique
  uniqueId    String   @unique  // EDU-SCH001-TCH-00001
  schoolId    String
  school      School   @relation(fields: [schoolId], references: [id])
  email       String
  firstName   String
  lastName    String
  role        UserRole
  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())
  
  @@index([schoolId])
}

model Student {
  id            String  @id @default(uuid())
  schoolId      String
  userId        String  @unique  // linked User record
  admissionNo   String
  classId       String
  isBoarder     Boolean @default(false)
  parentIds     String[]  // array of User IDs with Parent role
  
  @@index([schoolId])
  @@index([classId])
}

model Class {
  id            String @id @default(uuid())
  schoolId      String
  name          String   // "Grade 5A" | "Year 10B"
  curriculumType CurriculumType
  gradeLevel    String   // "Grade 5" | "Year 10"
  stream        String?  // "A" | "B"
  classTeacherId String?
  
  @@index([schoolId])
}

model Term {
  id          String   @id @default(uuid())
  schoolId    String
  name        String   // "Term 1 2026"
  startDate   DateTime
  endDate     DateTime
  isActive    Boolean  @default(false)
  
  @@index([schoolId])
}

model FeePayment {
  id              String   @id @default(uuid())
  schoolId        String
  studentId       String
  termId          String
  amount          Decimal
  currency        String   @default("KES")
  paymentMethod   PaymentMethod  // MPESA | BANK | CASH
  mpesaReceiptNo  String?
  stripePaymentId String?
  status          PaymentStatus
  paidAt          DateTime?
  createdAt       DateTime @default(now())
  
  @@index([schoolId])
  @@index([studentId])
}

model ProcurementRequest {
  id              String   @id @default(uuid())
  schoolId        String
  requestedById   String   // any staff role
  department      Department
  items           Json     // [{name, qty, estimatedCost, unit}]
  totalEstimate   Decimal
  status          ApprovalStatus
  approvalChain   Json     // [{role, userId, status, timestamp}]
  createdAt       DateTime @default(now())
  
  @@index([schoolId])
}

model AuditLog {
  id          String   @id @default(uuid())
  schoolId    String
  actorId     String
  action      String   // "exam.results.approved"
  entityType  String   // "ExamResult"
  entityId    String
  metadata    Json?
  createdAt   DateTime @default(now())
  
  @@index([schoolId])
  @@index([createdAt])
}
```

Full schema: `prisma/schema.prisma` (Sprint 002 deliverable).

---

## Payment Architecture

### Mpesa Fee Payment Flow (Parent → School)

```
1. Parent clicks "Pay Fees" in app
2. tRPC mutation: financial.initiateFeepayment({ studentId, amount, phoneNo })
3. Server calls Daraja STK Push API
4. Mpesa sends prompt to parent's phone
5. Parent enters PIN
6. Safaricom sends C2B callback to /api/webhooks/mpesa
7. Webhook validates signature
8. Inngest job triggered: processMpesaPayment({ receiptNo, amount, phone })
9. Job creates FeePayment record (idempotent — checks receiptNo exists)
10. Job updates student fee balance
11. Notification sent to parent + accountant
12. PDF receipt generated + emailed via Resend
```

### Stripe Subscription Flow (School → EduManage)

```
1. Super Admin / School Admin chooses tier
2. Stripe Checkout session created
3. School completes payment
4. Stripe webhook → /api/webhooks/stripe
5. Subscription record updated
6. Feature flags updated via Vercel Edge Config
```

---

## AI Architecture (Sprint 007)

All AI calls are async (Inngest jobs) to avoid Vercel 60s timeout:

```
Trigger (scheduled or user-initiated) →
  Inngest job queued →
  Job fetches relevant DB data →
  Formats prompt with school context →
  Calls Gemini 1.5 Pro API →
  Parses response →
  Saves AI-generated report/insight to DB →
  Notifies user: "Your report is ready"
```

**AI use cases by sprint 007:**
- Narrative term report generation (Dean/Headteacher)
- Fee defaulter analysis + payment pattern prediction
- Attendance anomaly detection (flag unusual absences)
- Timetable conflict detection
- Procurement spend category summaries
- Student academic progress narrative

---

## Notification Architecture

```
Event occurs (exam approved, fee paid, message received) →
  Inngest event emitted →
  Notification job runs:
    1. In-app notification created (NotificationLog table)
    2. Email sent via Resend (if preference enabled)
    3. (Future v2: SMS via Africa's Talking)
```

---

## Security Controls

- All API routes: Clerk JWT validation in middleware
- tRPC procedures: role-checked via `protectedProcedure(allowedRoles)`
- Database: RLS policies on all tables
- S3: no public buckets; presigned URLs only; 15-minute expiry
- Exam papers: separate S3 prefix + server-side encryption + time-locked access
- Webhooks: signature verification (Mpesa + Stripe)
- PII: no PII in logs; AuditLog stores entity IDs only
- HTTPS: enforced by Vercel
- Rate limiting: Vercel Edge middleware (100 req/min per IP)
- Input validation: Zod schemas on all tRPC inputs
