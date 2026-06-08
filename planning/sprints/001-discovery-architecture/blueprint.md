# Blueprint — Sprint 001: Discovery & Architecture

**Project:** EduManage  
**Sprint:** 001  
**Architectural pattern:** Feature-Sliced Clean Architecture + RLS Multi-Tenancy  
**Status:** Complete  
**Last updated:** 2026-05-29

---

## Sprint Scope Summary

This sprint is the architecture and planning sprint. No application code is produced. Deliverables are: CONTEXT.md, AGENTS.md, DOMAIN.md, DECISIONS.md, RISKS.md, QUESTIONS.md, STATE.md, ARCHITECTURE.md, DATABASE_SCHEMA outline, and this blueprint. The Builder begins work at Sprint 002.

---

## Technology Stack

| Concern | Choice | Justification |
|---|---|---|
| Frontend framework | Next.js 14 (App Router) | RSC for performance, collocated API routes, Vercel-native |
| Language | TypeScript strict | Type safety across 16 roles, reduces runtime errors |
| Styling | Tailwind CSS + shadcn/ui | Rapid UI, accessible components, consistent design system |
| Internal API | tRPC v11 | End-to-end type safety, no REST boilerplate |
| Database | Neon Postgres (serverless) | Prisma-native, branching for migrations, scale-to-zero |
| ORM | Prisma 5 | Type-safe queries, migration tooling, RLS-compatible |
| Auth | Clerk | Multi-org model = school tenants, role metadata on JWT |
| Async jobs | Inngest | Vercel-compatible, durable workflows, retry logic |
| File storage | AWS S3 (af-south-1) | Cost-effective at volume, presigned URLs, encryption |
| AI | Google Gemini 1.5 Pro | Operator preference, long context for school documents |
| Payments | Daraja Mpesa + Stripe | Mpesa for KE market, Stripe for international |
| Email | Resend | Developer-friendly, React Email templates |
| Monitoring | Sentry + Vercel Analytics | Error tracking + performance |
| Charts | Recharts | React-native, good for dashboards |
| Forms | React Hook Form + Zod | Validation collocated with schema |

---

## Architectural Pattern

**Pattern:** Feature-Sliced Clean Architecture with RLS Multi-Tenancy

The application is organised by feature domain (academic, financial, procurement, etc.) rather than by technical layer. Each feature owns its tRPC router, domain logic, and UI components. Cross-cutting concerns (auth, db, notifications) live in shared `lib/` and `server/` directories.

### Layer Structure

| Layer | Responsibility | Examples |
|---|---|---|
| Domain | Pure business logic, no framework deps | Fee calculation, attendance rules, approval chain logic |
| Application | Orchestrates domain + infrastructure | tRPC routers, Inngest jobs |
| Infrastructure | External systems | Prisma, S3 client, Mpesa client, Gemini client |
| Presentation | UI components, pages | Next.js pages, React components |

**Dependency rule:** Domain layer imports nothing. Application imports domain + infrastructure. Presentation imports application via tRPC.

---

## Multi-Tenancy Enforcement

Every table: `school_id UUID NOT NULL REFERENCES schools(id)`  
RLS policy template:
```sql
ALTER TABLE [table] ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON [table]
  USING (school_id = current_setting('app.current_school_id')::uuid);
```

Prisma client wrapper sets `app.current_school_id` on every connection from the tRPC context.

---

## Sprint 001 Deliverables Checklist

- [x] CONTEXT.md
- [x] AGENTS.md  
- [x] planning/DOMAIN.md
- [x] planning/DECISIONS.md (DEC-001 through DEC-008)
- [x] planning/RISKS.md (RISK-001 through RISK-008)
- [x] planning/QUESTIONS.md (Q-001 through Q-006)
- [x] planning/STATE.md
- [x] docs/ARCHITECTURE.md
- [x] sprints/001-discovery-architecture/blueprint.md (this file)
- [x] sprints/002-auth-multitenancy/requirements.md
- [x] sprints/002-auth-multitenancy/blueprint.md
- [x] sprints/002-auth-multitenancy/acceptance.md
- [x] sprints/002-auth-multitenancy/handoff-prompt.md
- [x] Full sprint roadmap (002–008 requirements)
