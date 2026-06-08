# Builder Handoff — Sprint 007: AI Automation Layer

Paste everything below into Claude Code. Sprint 006 must be fully passing.

---

You are building Sprint 007 of EduManage — the AI automation layer powered by Google Gemini 1.5 Pro. All features are Premium tier only. No AI content is ever auto-published.

## Pre-read (mandatory, in order):
1. `AGENTS.md` + `CONTEXT.md`
2. `planning/DECISIONS.md` (DEC-005 Gemini, DEC-015 quiz types, DEC-017 agents)
3. `planning/RISKS.md` (RISK-004 scope creep — AI must stay advisory only)
4. `planning/sprints/007-ai-automation/requirements.md`
5. `planning/sprints/007-ai-automation/blueprint.md`
6. `planning/sprints/007-ai-automation/acceptance.md`

## Agent routing:
- Gemini client, tRPC `ai.*` router, premiumProcedure middleware → `.claude/agents/nextjs-expert.md`
- AI banner component, report review UI, insights dashboard, feedback picker → `.claude/agents/react-frontend.md`
- Premium tier gate enforcement, role guards on ai.* procedures → `.claude/agents/clerk-auth.md`
- Inngest AI jobs (batch reports, anomaly detection, feedback), cron scheduling → `.claude/agents/devops.md`

## Summarise before coding — output:
1. New files (full paths)
2. Modified files + what changes
3. Prisma migration name
4. All Inngest job names + triggers
5. New env vars needed
6. Any `// ARCH-QUESTION:` items

Wait for operator approval. Then build.

## Implementation sequence:

### Phase 1: Foundation
1. `prisma migrate dev --name sprint_007_ai`
2. RLS SQL: `prisma/migrations/manual/006_rls_sprint007.sql`
3. `lib/ai/gemini.ts` — Gemini client with usage logging (blueprint.md pattern)
4. `server/trpc.ts` — add `premiumProcedure` middleware
5. `components/ai/AIBanner.tsx` — reusable banner component

### Phase 2: tRPC Router
6. `server/routers/ai.ts` — all procedures from blueprint.md (all use `premiumProcedure`)
7. Add `ai` router to `server/routers/index.ts`

### Phase 3: Inngest Jobs
8. `inngest/functions/ai-term-reports.ts` — batch with `concurrency: { limit: 5 }`
9. `inngest/functions/ai-anomaly-detection.ts` — cron `0 21 * * 1-5`, 4 detectors, upsert with @@unique
10. `inngest/functions/ai-feedback-suggestions.ts` — JSON parse with retry
11. `inngest/functions/ai-timetable-review.ts` — structured issues output
12. `inngest/functions/ai-lesson-plan.ts` — structured plan sections
13. `inngest/functions/ai-financial-narrative.ts`
14. `inngest/functions/ai-feedback-cleanup.ts` — nightly, delete expired AIFeedbackSuggestion rows

### Phase 4: UI
15. AI Reports page — review queue, edit, approve flow with AIBanner
16. AI Insights panel — unacknowledged insights, severity badges, acknowledge action
17. Assignment marking — "Suggest Feedback" button + option picker (non-blocking)
18. Timetable designer — "AI Review" button + issues sidebar
19. Lesson plan creator — "Generate Draft" button + loading state
20. AI Usage dashboard — token charts, feature breakdown (Recharts)

### Phase 5: Tests
21. Premium gate: Standard school → 403, no Gemini call made (mock Gemini client)
22. Anomaly idempotency: run detector twice, only one AIInsight created
23. Term report: batch of 5 students all complete within timeout
24. Feedback JSON parse: malformed Gemini response handled gracefully (retry + fallback)

## Critical rules — non-negotiable:
1. **ALL Gemini calls via Inngest** — never synchronous in tRPC; Vercel times out at 60s
2. **premiumProcedure on every `ai.*` procedure** — no exceptions
3. **AIBanner on every AI-generated content display** — no exceptions  
4. **Human approval gate** — `ai.reports.publish` blocked unless `status = APPROVED` by Headteacher
5. **Token logging** — every call logged, success AND failure — needed for billing
6. **Concurrency limit 5** — on term report batch job to avoid Gemini rate limits
7. **Feedback TTL** — `AIFeedbackSuggestion.expiresAt = now() + 24h`; cleanup job runs nightly
8. **JSON.parse in try/catch** — Gemini sometimes returns markdown-wrapped JSON; strip fences before parsing
9. **Anomaly uniqueness** — @@unique DB constraint is source of truth, not application code

## New env var:
```env
GOOGLE_GENERATIVE_AI_API_KEY=
```

## Install:
```bash
npm install @google/generative-ai
```

## Definition of done:
All AC-001 through AC-005 pass. Premium gate verified. Anomaly idempotency verified. No AI content published without HT approval. Token logging verified for success + failure. CONTEXT.md + STATE.md updated.
