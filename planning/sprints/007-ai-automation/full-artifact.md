# Sprint 007: AI Automation Layer — Full Artifact

**Project:** EduManage  
**Sprint:** 007  
**Status:** Approved  
**Last updated:** 2026-05-29  
**Depends on:** Sprint 006 complete  
**Tier gate:** Premium tier only (DEC-005, DEC-015)

---

# Requirements

## Sprint Goal
Premium-tier schools receive AI-powered automation: narrative report generation, anomaly detection, smart assignment feedback, timetable conflict detection, procurement spend analysis, and a lesson plan assistant — all powered by Google Gemini 1.5 Pro running asynchronously via Inngest.

**All AI features are Premium tier only. All AI outputs are advisory — no autonomous actions.**

---

## FR-001: Narrative Term Report Generation

**Roles:** Dean initiates; Headteacher reviews + publishes

- Dean triggers "Generate Term Reports" for a class
- Inngest job: for each student, fetches exam results, quiz scores, attendance %, assignment completion, teacher comments
- Gemini 1.5 Pro generates a 200–400 word narrative per student
- Dean + Class Teacher review, edit if needed, then Headteacher approves
- Published report delivered to parent in-app + email

**Prompt template:**
```
You are writing an end-of-term academic report for [student name], [grade], at [school name].
Data: Exam average: [X]%, Attendance: [Y]%, Assignments: [Z]% on time, Quiz average: [W]%.
Subject performances: [subject: grade, teacher comment].
Write a warm, constructive, professional report (200-400 words). Highlight strengths first, then areas for growth. End with an encouraging statement. Do not mention specific scores — use qualitative language.
Output ONLY the report text. No preamble.
```

**Data:**
```
AIReport {
  id, schoolId, reportType (TERM_REPORT | FINANCIAL_SUMMARY | DEPT_SUMMARY),
  entityId, entityType (STUDENT | SCHOOL | DEPARTMENT),
  termId?, prompt, rawOutput, editedOutput?,
  status (GENERATED | REVIEWED | APPROVED | PUBLISHED),
  generatedAt, reviewedByUserId?, approvedByUserId?, publishedAt?,
  tokensUsed, modelVersion
}
```

---

## FR-002: Anomaly Detection (Nightly Inngest Jobs)

Four detectors run nightly at 21:00 KE on all Premium schools:

| Detector | Condition | Notification to |
|---|---|---|
| Attendance anomaly | Student >20% absence rate this term | Class Teacher + Dean |
| Fee defaulter | Student >50% outstanding balance at week 6 | Accountant + Headteacher |
| Score drop | Student score drops >20 percentage points from last term | Class Teacher + Dean |
| Procurement overspend | Department spending >110% of approved budget | Accountant |

Each anomaly creates an `AIInsight` record and an in-app notification.

**Data:**
```
AIInsight {
  id, schoolId, insightType, entityId, entityType, severity (LOW|MEDIUM|HIGH),
  title, description, recommendedAction?, isAcknowledged, acknowledgedByUserId?,
  detectedAt, createdAt
}
```

---

## FR-003: Smart Assignment Feedback Suggestions

**Roles:** Teacher (on marking page)

- After teacher enters a score, a "Suggest Feedback" button appears
- On click: Inngest job fires → Gemini generates 2–3 feedback options
- Teacher selects one, edits if needed, saves as final feedback
- AI suggestion is non-blocking — teacher can write own feedback and ignore

**Prompt:**
```
Assignment: [title]. Student score: [X]/[max]. 
Write 2-3 distinct constructive feedback options (2-3 sentences each). 
Options should vary in tone (encouraging, direct, detailed). 
Output as JSON array: [{"label":"Encouraging","text":"..."},...]
```

---

## FR-004: Timetable Conflict Detection (Real-time)

**Roles:** Dean (on timetable designer)

- When Dean saves any timetable slot, conflict check runs immediately (deterministic, not AI)
- When Dean clicks "AI Review Timetable", Gemini analyses full timetable for:
  - Teacher workload imbalance (>5 periods/day)
  - Students with >2 consecutive same-subject periods
  - No breaks scheduled
  - Classes with fewer than required periods per subject per week (CBC/IGCSE minimums)
- AI returns structured JSON with issues + suggestions
- Dean sees flagged issues in a sidebar

---

## FR-005: Lesson Plan Assistant

**Roles:** Teacher (on lesson plan creation page)

- Teacher types: subject, class, topic, duration, learning objectives
- Clicks "Generate Draft" → Inngest job → Gemini generates full lesson plan structure:
  - Introduction (5 min)
  - Main activity (broken into steps)
  - Assessment activity
  - Resources needed
  - Homework suggestion
- Teacher reviews, edits, saves as their lesson plan
- Download as .docx via existing lesson plan export

---

## FR-006: Financial AI Narrative

**Roles:** Accountant triggers; Headteacher views

- After generating a financial report (Sprint 004), Accountant can click "Generate AI Summary"
- Gemini writes a 150–300 word executive summary of the financial report
- Summary included in the PDF report + shown on Headteacher's financial dashboard

---

## FR-007: AI Usage Controls & Logging

- All AI calls: logged in `AIUsageLog` with token counts
- Premium schools: unlimited (within Gemini API rate limits)
- AI outputs: always show banner "AI-generated — please review before publishing"
- School-level AI usage dashboard (IT Support / Headteacher): tokens used, reports generated, insights raised

**Data:**
```
AIUsageLog {
  id, schoolId, userId, feature, promptTokens, completionTokens,
  totalTokens, modelVersion, latencyMs, success, errorMessage?, createdAt
}
```

---

# Blueprint

## Gemini Integration

```typescript
// lib/ai/gemini.ts
import { GoogleGenerativeAI } from '@google/generative-ai'

const genAI = new GoogleGenerativeAI(process.env.GOOGLE_GENERATIVE_AI_API_KEY!)

export async function generateWithGemini(prompt: string, schoolId: string, feature: string): Promise<{
  text: string
  tokensUsed: number
}> {
  const model = genAI.getGenerativeModel({ model: 'gemini-1.5-pro' })
  const startMs = Date.now()
  
  try {
    const result = await model.generateContent(prompt)
    const text = result.response.text()
    const usage = result.response.usageMetadata
    
    // Log usage (fire and forget)
    void logAIUsage({
      schoolId, feature,
      promptTokens: usage?.promptTokenCount ?? 0,
      completionTokens: usage?.candidatesTokenCount ?? 0,
      latencyMs: Date.now() - startMs,
      success: true
    })
    
    return { text, tokensUsed: (usage?.totalTokenCount ?? 0) }
  } catch (err) {
    void logAIUsage({ schoolId, feature, success: false, errorMessage: String(err), latencyMs: Date.now() - startMs })
    throw err
  }
}
```

## Premium Tier Guard

```typescript
// server/middleware/premiumOnly.ts
export const premiumProcedure = protectedProcedure([...ALL_ROLES])
  .use(async ({ ctx, next }) => {
    const sub = await ctx.prisma.schoolSubscription.findUnique({ where: { schoolId: ctx.schoolId } })
    if (sub?.tier !== 'PREMIUM') {
      throw new TRPCError({ code: 'FORBIDDEN', message: 'This feature requires a Premium subscription.' })
    }
    return next()
  })
```

## Inngest Jobs

| Job | Trigger | Action |
|---|---|---|
| `generateTermReports` | `ai/term-reports.requested` | Batch: fetch data per student → Gemini → save AIReport rows |
| `detectAnomalies` | Cron: nightly 21:00 | Run 4 detectors → create AIInsight + Notification rows |
| `suggestAssignmentFeedback` | `ai/feedback.requested` | Gemini → return 3 options → save to temporary `AIFeedbackSuggestion` |
| `reviewTimetable` | `ai/timetable-review.requested` | Fetch timetable → Gemini analysis → return structured issues |
| `generateLessonPlan` | `ai/lesson-plan.requested` | Gemini → structured lesson plan → save to LessonPlan draft |
| `generateFinancialNarrative` | `ai/financial-narrative.requested` | Fetch financial report → Gemini → save to AIReport |

## New npm Packages

```bash
npm install @google/generative-ai
```

---

# Acceptance Criteria

## AC-001: Term Report Generation
- Dean triggers term reports for Class 7A (30 students) → 30 Inngest jobs queued
- Each completes within 60 seconds → 30 AIReport rows with `status = GENERATED`
- Dean views report for Student Achieng → sees 200–400 word narrative + "AI-generated — review before publishing" banner
- Headteacher approves → parent receives in-app notification + email with report

## AC-002: Anomaly Detection
- Student with 25% attendance triggers attendance anomaly detector → AIInsight created + Class Teacher notified
- Student with same attendance on a Basic-tier school → NO insight created (Premium gate enforced)

## AC-003: Assignment Feedback
- Teacher clicks "Suggest Feedback" on marked submission → 3 feedback options returned within 10 seconds
- Teacher selects option 2, edits one sentence, saves → saved feedback is the edited version, not the raw AI output
- AI feedback banner present on all AI-generated content

## AC-004: Lesson Plan Assistant
- Teacher inputs: Maths, Grade 8, "Quadratic Equations", 40 min → draft lesson plan generated with all sections
- Teacher edits "Main Activity" section → saves → downloaded .docx contains the edited version

## AC-005: AI Usage Controls
- Non-premium school attempts AI endpoint → receives 403 with "This feature requires a Premium subscription"
- Premium school: AIUsageLog row created for every successful Gemini call with correct token counts

---

# Builder Handoff — Sprint 007

Paste into Claude Code. Sprint 006 must be passing.

**Pre-read:** all Sprint 007 planning files + DEC-005 (Gemini), DEC-017 (agents).

**Summarise before coding.** Wait for approval.

**Critical rules:**
- ALL Gemini calls MUST go through Inngest jobs — never in synchronous tRPC procedures (Vercel 60s timeout)
- ALL AI features MUST be gated with `premiumProcedure` middleware
- ALL AI outputs MUST include banner text "AI-generated — please review before publishing"
- Never auto-publish AI-generated content — always require human approval step
- Token usage MUST be logged for every call — usage monitoring is required
- Batch report generation: use Inngest's `step.run` with concurrency limit of 5 (avoid Gemini rate limits)
- AIInsight anomalies: idempotent — same anomaly for same student same term should not create duplicate

**Install:**
```bash
npm install @google/generative-ai
```

**Definition of done:** AC-001 through AC-005 pass. Premium gate tested. Token logging verified. No AI output published without human approval step. CONTEXT.md + STATE.md updated.
