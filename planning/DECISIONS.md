# DECISIONS.md — EduManage

Durable architecture and product decisions. Add new entries; never delete old ones.

---

## DEC-001: Multi-Tenancy Model — RLS over Schema-per-Tenant

**Date:** 2026-05-29  
**Made by:** Architect + Operator  
**Context:** Platform must isolate school data. Three options exist: RLS, schema-per-tenant, db-per-tenant.  
**Options considered:**
- RLS: school_id on every table, Postgres RLS policies — simple, cheap, scales to 500+ schools without ops overhead
- Schema-per-tenant: stronger isolation, but Prisma support is immature and migration management becomes complex at scale
- DB-per-tenant: maximum isolation, but unmanageable cost and ops for a bootstrapped team of 4  
**Decision:** Row-Level Security (RLS) with school_id on every table  
**Rationale:** Team size and budget make schema/db-per-tenant untenable. RLS with Neon Postgres is well-understood and sufficient for Kenya-first launch of ≤500 schools.  
**Consequences:** Every query MUST include school_id filter. All Prisma queries scoped through a school-aware client wrapper. Penetration testing required before first school onboarding.

---

## DEC-002: Authentication — Clerk over Supabase Auth

**Date:** 2026-05-29  
**Made by:** Architect + Operator  
**Context:** Auth must support: multi-tenant orgs, role-based access, school-assigned emails, 16 distinct roles.  
**Options considered:**
- Clerk: built-in org/membership model maps directly to school tenants; role metadata on JWT; excellent Next.js integration; school-email enforcement via allowlist
- Supabase Auth: tightly coupled to Supabase DB; switching to Neon + Prisma makes Supabase Auth redundant; row-level user management less flexible  
**Decision:** Clerk  
**Rationale:** Clerk's Organisation model maps exactly to a school tenant. Metadata on JWT carries role + school_id, enabling server-side RLS without extra DB round trips.  
**Consequences:** Clerk per-MAU pricing becomes a cost factor at 10,000+ users. Revisit at 5,000 MAU.

---

## DEC-003: Database — Neon (serverless Postgres) over Supabase DB

**Date:** 2026-05-29  
**Made by:** Architect + Operator  
**Context:** Needed a Postgres host compatible with Prisma ORM and serverless/Vercel deployment.  
**Options considered:**
- Neon: serverless Postgres, branching for dev/test, Prisma-native, scale-to-zero on free tier
- Supabase: includes auth + storage (but we're using Clerk + S3), creates unnecessary coupling; realtime not needed  
**Decision:** Neon  
**Rationale:** Clean Prisma integration, database branching enables safe sprint-by-sprint migrations, no bundled services we don't use.  
**Consequences:** No Supabase realtime. WebSockets for live notifications handled via Inngest events + polling or Pusher (deferred to Sprint 005).

---

## DEC-004: File Storage — AWS S3

**Date:** 2026-05-29  
**Made by:** Architect + Operator  
**Context:** Platform needs to store exam papers (.docx), lesson plans, library resources (.pdf, .docx, links), student assignments.  
**Options considered:**
- Vercel Blob: convenient but 5GB free limit; expensive at scale; limited control
- AWS S3: industry standard, cheap at volume, presigned URL pattern well-understood, lifecycle policies for archiving
- GCP Cloud Storage: excellent but adds second cloud vendor; no advantage over S3 for this stack  
**Decision:** AWS S3 (region: af-south-1 Cape Town preferred for data residency; fallback eu-west-1)  
**Rationale:** Cost, maturity, presigned URL security model for exam papers, lifecycle policies for storage quota enforcement.  
**Consequences:** AWS account required. IAM roles must be scoped per-school bucket prefix. Operator to confirm region.

---

## DEC-005: AI Provider — Google Gemini

**Date:** 2026-05-29  
**Made by:** Operator  
**Context:** Platform requires AI for report generation, anomaly detection, timetable conflict detection, spend summaries.  
**Options considered:**
- Google Gemini 1.5 Pro: strong document understanding, multimodal, competitive pricing
- OpenAI GPT-4o: industry standard but operator preference is Gemini
- No AI in v1: deferred to Sprint 007, after core platform stable  
**Decision:** Google Gemini 1.5 Pro  
**Rationale:** Operator preference. Gemini 1.5 Pro's long context window handles full school financial reports and attendance histories in a single prompt.  
**Consequences:** Google AI Studio API key required. AI features gated to Premium tier only. All AI calls routed through Inngest jobs (not synchronous API routes) to avoid Vercel timeout limits.

---

## DEC-006: Payment Rails — Mpesa + Stripe Dual

**Date:** 2026-05-29  
**Made by:** Architect + Operator  
**Context:** Schools pay EduManage subscriptions; parents pay school fees through the platform.  
**Options considered:**
- Mpesa only: covers Kenya but blocks international schools
- Stripe only: covers international but useless for Kenyan parents
- Both: correct split — Mpesa for parent fee payments + local school subscriptions; Stripe for international school subscriptions  
**Decision:** Daraja Mpesa API (STK Push + C2B) for parent payments + local subscriptions; Stripe for international subscriptions  
**Rationale:** Mpesa penetration in Kenya is near-universal for school fee payments. Stripe handles card billing for diaspora/international schools.  
**Consequences:** Daraja sandbox + production credentials required. Mpesa callbacks must be idempotent. Stripe webhook handler required. Two separate payment reconciliation flows.

---

## DEC-007: Internal API Layer — tRPC over REST

**Date:** 2026-05-29  
**Made by:** Architect  
**Context:** Next.js 14 fullstack app needs type-safe API layer between frontend and backend.  
**Options considered:**
- REST via Next.js API routes: flexible but no type safety end-to-end; boilerplate heavy
- tRPC: end-to-end TypeScript type safety; zero boilerplate for client-server contracts; excellent Next.js App Router support
- GraphQL: overkill for a team of 2 developers; steep learning curve  
**Decision:** tRPC with Next.js App Router  
**Rationale:** Team of 4 (2 devs effective) benefits most from compiler-enforced contracts. tRPC eliminates an entire class of runtime type errors.  
**Consequences:** All internal API calls through tRPC routers. External webhooks (Mpesa, Stripe) use standard Next.js API routes (not tRPC).

---

## DEC-008: Sprint Delivery Order

**Date:** 2026-05-29  
**Made by:** Architect  
**Context:** 8 major sprint areas identified. Order must maximise value delivery and unblock subsequent sprints.  
**Decision:** 001 Architecture → 002 Auth/Tenancy → 003 Academic Core → 004 Financial/Mpesa → 005 Communications → 006 Library/Resources → 007 AI Layer → 008 Boarding/Transport/Support Depts  
**Rationale:** Auth and tenancy must exist before any feature. Academic core is the platform's primary value. Financial/Mpesa enables revenue. Communications and library are high-value additions. AI and support departments are Sprint 1 premium differentiators but depend on core data existing.  
**Consequences:** Schools can be onboarded and charged after Sprint 004. Full platform complete at Sprint 008.

---

## DEC-009: KRA Tax — Report Generation Only (No iTax API)

**Date:** 2026-05-29  
**Made by:** Operator  
**Decision:** Generate KRA-formatted PDF/CSV reports for manual iTax upload. No direct API.  
**Rationale:** iTax API is unstable and requires KRA developer program enrollment. Risk too high for v1.  
**Consequences:** Accountants download reports and upload manually to iTax. Add iTax API to v2 backlog.

---

## DEC-010: Storage Quotas — School-Wide Tier-Based

**Date:** 2026-05-29  
**Made by:** Operator  
**Decision:** Basic: 50GB / Standard: 200GB / Premium: 1TB per school.  
**Consequences:** StorageUsage table tracks per-school usage. S3 lifecycle policies archive old files. No per-user enforcement in v1.

---

## DEC-011: Exam Papers — Encrypted + Time-Locked in S3

**Date:** 2026-05-29  
**Made by:** Operator  
**Decision:** Exam papers uploaded by teachers, encrypted at rest in S3, released only after Dean sets a release timestamp.  
**Consequences:** ExamPaper entity has `releasedAt` field. Presigned URL generation blocked until `now() >= releasedAt`. Dean-only release controls.

---

## DEC-012: Attendance — Manual Digital Only (v1)

**Date:** 2026-05-29  
**Made by:** Operator  
**Decision:** Class teachers mark attendance manually in the app. No biometric hardware.  
**Consequences:** Simple, fast Sprint 003 implementation. Hardware adapter interface documented for v2.

---

## DEC-013: File Storage — AWS S3 eu-west-1 Primary

**Date:** 2026-05-29  
**Made by:** Operator  
**Decision:** S3 eu-west-1 (Ireland) for v1. GCP Cloud Storage europe-west1 evaluated for v2 replication.  
**Consequences:** Single AWS account. No multi-cloud complexity in v1. S3 bucket names prefixed `edumanage-[env]-`.

---

## DEC-014: Salary Disbursement — Manual via School Bank

**Date:** 2026-05-29  
**Made by:** Operator  
**Decision:** EduManage generates payslips + PAYE calculations only. School disburses salaries via their own bank manually.  
**Consequences:** No Mpesa B2C integration for payroll. Simpler compliance posture. Payslip PDF emailed to staff via Resend.

---

## DEC-015: Quiz Engine — Question Types

**Date:** 2026-05-29  
**Made by:** Architect (pending operator confirmation)  
**Decision:** v1: Multiple Choice + True/False (auto-graded) + Short Answer (manually graded by teacher). Essay type added in Sprint 007 with Gemini AI grading.  
**Consequences:** QuizQuestion has `type` enum: MCQ | TRUE_FALSE | SHORT_ANSWER | ESSAY. Auto-grading runs only for MCQ and TRUE_FALSE. SHORT_ANSWER flags for teacher review.

---

## DEC-016: Analytics — Full History Retained, Monthly Rollups After 2 Years

**Date:** 2026-05-29  
**Made by:** Architect  
**Decision:** Raw analytics events retained for 2 years. Monthly summaries computed and raw data archived to S3 after 2 years.  
**Consequences:** AnalyticsEvent table partitioned by month. Inngest job runs monthly rollup. Charts query summary tables for data >2 years old.

---

## DEC-017: Agent Architecture — 4 Specialised Builder Agents

**Date:** 2026-05-29  
**Made by:** Architect  
**Decision:** Four `.claude/agents` files created: nextjs-expert, react-frontend, devops, clerk-auth. Each agent has domain-specific instructions, tools, and rules. Builder routes tasks to the correct agent.  
**Consequences:** Faster, more accurate code generation. Each agent has role-specific constraints. Operator selects agent based on task type.
