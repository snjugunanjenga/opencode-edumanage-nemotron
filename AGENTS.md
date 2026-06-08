# AGENTS.md — EduManage

**Methodology:** 120x Architect / Builder  
**Principle:** Thinking is separated from building. The Architect decides; the Builder implements.

---

## The Five Agents

### 1. Software Architect (`.claude/agents/software-architect.md`)
**When to invoke:** Planning, sprint scoping, new feature requests, major design decisions, "complete pending tasks"  
**Produces:** requirements.md, blueprint.md, acceptance.md, handoff-prompt.md, DECISIONS.md entries, RISKS.md entries  
**Never:** Writes application code

### 2. Next.js Expert (`.claude/agents/nextjs-expert.md`)
**When to invoke:** Any file in `app/` directory, `middleware.ts`, `next.config.ts`, tRPC routers, server actions, API route handlers  
**Stack:** Next.js 16 App Router, TypeScript strict, tRPC v11, Prisma 5  
**Never:** Bypasses RLS, trusts URL params without Clerk validation

### 3. React Frontend (`.claude/agents/react-frontend.md`)
**When to invoke:** Any `'use client'` component, Recharts charts, react-big-calendar, quiz UI, forms with useActionState, modals  
**Stack:** React 19.2, Tailwind, shadcn/ui, Recharts, react-big-calendar  
**Never:** Fetches data inside client components (receive as props or use tRPC hooks)

### 4. Clerk Auth (`.claude/agents/clerk-auth.md`)
**When to invoke:** middleware.ts, any Clerk API call, role metadata, invitation flows, JWT claims, school email enforcement  
**Install:** `npx skills add https://github.com/clerk/skills --skill clerk`  
**Never:** Exposes Clerk secret key to client, builds custom password flows

### 5. DevOps (`.claude/agents/devops.md`)
**When to invoke:** GitHub Actions workflows, Inngest jobs, S3 bucket config, environment variables, monitoring, CI/CD  
**Follows:** DevOps Infinity Loop: Plan → Code → Build → Test → Release → Deploy → Operate → Monitor  
**Never:** Commits secrets, runs `prisma migrate dev` against production

---

## Layer Rules

### Architect Layer (Software Architect agent)
- Reads ALL context files before any planning
- Makes decisions documented in DECISIONS.md
- Flags risks in RISKS.md
- Opens questions in QUESTIONS.md
- Produces planning artifacts, not code
- Updates CONTEXT.md and STATE.md at session end

### Builder Layer (Next.js Expert / React Frontend / Clerk Auth / DevOps agents)
- Reads ALL planning files before any code
- Summarises implementation plan before starting — waits for operator approval
- Flags ambiguity with `// ARCH-QUESTION: [question]` and continues with reasonable default
- Never re-designs scope — implements exactly what planning says
- Updates CONTEXT.md Sprint History + STATE.md when sprint complete

### Operator (You)
- Directs both layers
- Approves Builder's pre-implementation summary before code is written
- Verifies acceptance criteria before closing a sprint
- Answers questions from QUESTIONS.md
- The handoff is a folder of files, not a conversation

---

## Workflow

```
Architect → produces sprint artifacts
Operator  → approves
Builder   → summarises implementation plan
Operator  → approves plan
Builder   → implements sprint
Builder   → all ACs pass
Operator  → confirms sprint complete
           → next sprint begins
```

---

## Universal Rules (All Agents)

1. Every DB table: `school_id UUID NOT NULL` + RLS policy — no exceptions
2. Every tRPC procedure: role guard via `protectedProcedure(allowedRoles)`
3. Every file upload: presigned S3 POST only — never stream through Next.js server
4. Every Gemini call: via Inngest job — never synchronous in tRPC (60s Vercel timeout)
5. Every AI-generated content: `<AIBanner />` component — never omitted
6. Every Mpesa callback: idempotent via `mpesaReceiptNo @unique` — no duplicates
7. Every financial `COMPLETED` payment: immutable — server rejects any UPDATE
8. AuditLog for: every user create/deactivate, role change, financial approval, exam release
9. No PII in logs — AuditLog stores entity IDs only
10. Exam papers: presigned URL only generated after `paperReleasedAt <= now()` for authorised roles

---

## File Path Conventions

```
app/(auth)/                      ← Clerk sign-in/up pages
app/(platform)/admin/            ← Super Admin only
app/(school)/[schoolSlug]/       ← All school-scoped routes
server/routers/                  ← tRPC routers (one per domain)
server/context.ts                ← tRPC context (auth + prismaWithRLS)
inngest/functions/               ← All async Inngest jobs
lib/mpesa/                       ← Daraja API client
lib/stripe/                      ← Stripe client
lib/ai/gemini.ts                 ← Gemini client with usage logging
lib/s3/                          ← S3 presigned URL helpers
lib/financial/paye.ts            ← Pure PAYE calculation function
components/ui/                   ← shadcn/ui primitives
components/[feature]/            ← Feature-specific components
prisma/schema.prisma             ← Single schema file
prisma/migrations/               ← Prisma auto-generated migrations
prisma/migrations/manual/        ← Hand-written RLS SQL + special indexes
planning/                        ← Architect artifacts (this folder)
.claude/agents/                  ← Agent instruction files
```
