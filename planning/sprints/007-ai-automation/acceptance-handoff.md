# Acceptance Criteria — Sprint 007: AI Automation Layer

**Last updated:** 2026-05-29

---

## AC-001: Term Report Generation (Premium Gate + Content)

**Given** a Premium school's Dean clicks "Generate Term Reports" for Class 7A (30 students)  
**When** the Inngest batch job runs  
**Then** 30 `AIReport` rows exist with `status = GENERATED` within 5 minutes; each has 200–400 words; no raw scores appear in text (qualitative language only)

**Given** a Standard-tier school's Dean hits the same endpoint  
**When** tRPC called  
**Then** returns `FORBIDDEN: "This feature requires a Premium subscription"` — no Gemini call made, no tokens consumed

**Given** Headteacher approves report for Student Achieng  
**When** `ai.reports.approve` called  
**Then** `AIReport.status = APPROVED`; parent receives in-app notification; parent can view report in their portal; "AI-generated — please review before publishing" banner visible

---

## AC-002: Anomaly Detection (Idempotency + Gating)

**Given** Student Kamau has 25% attendance this term (>20% threshold) on a Premium school  
**When** nightly anomaly detection Inngest job runs at 21:00  
**Then** `AIInsight` created with `insightType = ATTENDANCE_ANOMALY`, `entityId = kamauId`; Class Teacher + Dean notified

**Given** the same job runs again the following night (Kamau still at 25%)  
**When** job runs  
**Then** NO duplicate `AIInsight` created (@@unique constraint on `schoolId + insightType + entityId + termId`)

**Given** same student Kamau on a Standard-tier school  
**When** job runs  
**Then** no insight created for that school (Premium gate on anomaly detection)

---

## AC-003: Feedback Suggestions

**Given** Teacher marks submission with score 65/100 and clicks "Suggest Feedback"  
**When** Inngest job completes  
**Then** 3 feedback options returned (Encouraging / Direct / Detailed); each 2–3 sentences

**Given** Teacher selects option 2, edits one sentence, clicks Save  
**When** feedback saved  
**Then** `AssignmentSubmission.feedback` = the teacher's edited text (not raw AI output); AI banner NOT shown on saved feedback (human-edited content)

**Given** Teacher ignores AI suggestions and writes own feedback  
**When** saved  
**Then** own feedback saved normally; AI suggestions discarded silently

---

## AC-004: Lesson Plan Assistant

**Given** Teacher inputs: Maths, Grade 8A, "Quadratic Equations", 40 min, ["Solve quadratics by factoring", "Apply to word problems"]  
**When** "Generate Draft" clicked and Inngest job completes  
**Then** LessonPlan DRAFT created with all 5 sections (intro, main activity, assessment, resources, homework)

**Given** Teacher edits the "Main Activity" section  
**When** saved  
**Then** `LessonPlan.status = DRAFT`; downloaded .docx contains the teacher's edited version

---

## AC-005: AI Usage Logging

**Given** any Gemini call is made successfully  
**When** call completes  
**Then** `AIUsageLog` row created with: correct `schoolId`, `userId`, `feature`, `promptTokens`, `completionTokens`, `latencyMs`, `success = true`

**Given** a Gemini call fails (timeout or API error)  
**When** error occurs  
**Then** `AIUsageLog` row created with `success = false`, `errorMessage` populated; no partial report saved; Dean notified "Report generation failed for X students"

**Given** IT Support views the AI usage dashboard  
**When** page loads  
**Then** shows: total tokens consumed this month, breakdown by feature, number of reports generated; all scoped to their school

---

## Definition of Done

All AC-001 through AC-005 pass.  
Premium gate tested: Standard school gets 403, no Gemini calls made.  
Anomaly detection idempotency tested (run twice, one insight created).  
All AI outputs have banner component.  
No AI content auto-published (always requires human approval step).  
`AIUsageLog` verified for success and failure cases.  
Inngest jobs tested with `inngest-cli dev`.  
`CONTEXT.md` + `STATE.md` updated.

---

# Builder Handoff — Sprint 007

Paste into Claude Code. Sprint 006 must be passing.

## Pre-read (mandatory):
1. `AGENTS.md` + `CONTEXT.md`
2. `planning/DECISIONS.md` (DEC-005 Gemini, DEC-015 quiz types, DEC-017 agents)
3. `planning/sprints/007-ai-automation/requirements.md`
4. `planning/sprints/007-ai-automation/blueprint.md`
5. `planning/sprints/007-ai-automation/acceptance.md`

## Summarise before coding. Wait for approval.

## Critical rules:
1. **ALL Gemini calls through Inngest** — NEVER in synchronous tRPC (Vercel 60s timeout)
2. **Premium gate** — use `premiumProcedure` middleware on every `ai.*` router procedure
3. **AI banner** — `<AIBanner />` component on every page rendering AI-generated content — no exceptions
4. **Human approval required** — no `ai.reports.publish` without prior `ai.reports.approve` by Headteacher
5. **Token logging** — log every call, success AND failure — billing depends on it
6. **Batch concurrency** — Inngest `concurrency: { limit: 5 }` on term report generation — prevents Gemini rate limits
7. **Feedback suggestions TTL** — store in `AIFeedbackSuggestion` temp table with `expiresAt = now() + 24h`; Inngest cleanup job removes expired rows nightly
8. **JSON response parsing** — wrap `JSON.parse(result.text)` in try/catch; if Gemini returns invalid JSON, retry once with more explicit prompt
9. **Anomaly idempotency** — rely on DB `@@unique` constraint, not application-level deduplication

## Install:
```bash
npm install @google/generative-ai
```

## Definition of done: all AC-001 through AC-005 pass. CONTEXT.md + STATE.md updated.
