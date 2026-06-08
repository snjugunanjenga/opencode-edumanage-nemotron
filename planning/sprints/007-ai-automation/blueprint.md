# Blueprint — Sprint 007: AI Automation Layer

**Project:** EduManage  
**Sprint:** 007  
**Last updated:** 2026-05-29

---

## New Prisma Models

```prisma
model AIReport {
  id               String        @id @default(uuid())
  schoolId         String
  reportType       AIReportType
  entityId         String        // studentId | schoolId | departmentId
  entityType       String        // STUDENT | SCHOOL | DEPARTMENT
  termId           String?
  prompt           String        @db.Text
  rawOutput        String        @db.Text
  editedOutput     String?       @db.Text
  status           AIReportStatus @default(GENERATED)
  generatedAt      DateTime      @default(now())
  reviewedByUserId String?
  approvedByUserId String?
  publishedAt      DateTime?
  tokensUsed       Int
  modelVersion     String        @default("gemini-1.5-pro")
  @@index([schoolId, entityType, entityId])
  @@index([schoolId, status])
}

model AIInsight {
  id                   String          @id @default(uuid())
  schoolId             String
  insightType          String          // ATTENDANCE_ANOMALY | FEE_DEFAULTER | SCORE_DROP | PROCUREMENT_OVERSPEND
  entityId             String
  entityType           String          // STUDENT | DEPARTMENT
  severity             InsightSeverity @default(MEDIUM)
  title                String
  description          String
  recommendedAction    String?
  isAcknowledged       Boolean         @default(false)
  acknowledgedByUserId String?
  acknowledgedAt       DateTime?
  detectedAt           DateTime        @default(now())
  termId               String
  @@unique([schoolId, insightType, entityId, termId])  // idempotent per term
  @@index([schoolId, isAcknowledged])
}

model AIUsageLog {
  id               String   @id @default(uuid())
  schoolId         String
  userId           String
  feature          String   // TERM_REPORT | ANOMALY_DETECTION | FEEDBACK_SUGGESTION | TIMETABLE_REVIEW | LESSON_PLAN | FINANCIAL_NARRATIVE
  promptTokens     Int      @default(0)
  completionTokens Int      @default(0)
  totalTokens      Int      @default(0)
  modelVersion     String   @default("gemini-1.5-pro")
  latencyMs        Int
  success          Boolean
  errorMessage     String?
  createdAt        DateTime @default(now())
  @@index([schoolId, createdAt])
  @@index([schoolId, feature])
}

enum AIReportType   { TERM_REPORT FINANCIAL_SUMMARY DEPT_SUMMARY }
enum AIReportStatus { GENERATED REVIEWED APPROVED PUBLISHED }
enum InsightSeverity { LOW MEDIUM HIGH }
```

---

## Gemini Client

```typescript
// lib/ai/gemini.ts
import { GoogleGenerativeAI } from '@google/generative-ai'
import { inngest } from '@/inngest/client'

const genAI = new GoogleGenerativeAI(process.env.GOOGLE_GENERATIVE_AI_API_KEY!)

export interface GeminiResult {
  text: string
  promptTokens: number
  completionTokens: number
  totalTokens: number
}

export async function callGemini(
  prompt: string,
  options: { maxOutputTokens?: number; temperature?: number } = {}
): Promise<GeminiResult> {
  const model = genAI.getGenerativeModel({
    model: 'gemini-1.5-pro',
    generationConfig: {
      maxOutputTokens: options.maxOutputTokens ?? 1024,
      temperature: options.temperature ?? 0.7,
    },
  })

  const result = await model.generateContent(prompt)
  const text = result.response.text()
  const usage = result.response.usageMetadata

  return {
    text,
    promptTokens: usage?.promptTokenCount ?? 0,
    completionTokens: usage?.candidatesTokenCount ?? 0,
    totalTokens: usage?.totalTokenCount ?? 0,
  }
}
```

---

## Premium Procedure Middleware

```typescript
// server/trpc.ts (addition)
export const premiumProcedure = (allowedRoles: UserRole[]) =>
  protectedProcedure(allowedRoles).use(async ({ ctx, next }) => {
    const sub = await ctx.prisma.schoolSubscription.findUnique({
      where: { schoolId: ctx.schoolId },
      select: { tier: true, status: true },
    })
    if (sub?.tier !== 'PREMIUM' || sub?.status !== 'ACTIVE') {
      throw new TRPCError({
        code: 'FORBIDDEN',
        message: 'This feature requires a Premium subscription. Upgrade at /settings/subscription',
      })
    }
    return next()
  })
```

---

## tRPC Router: `ai.*`

```typescript
// All procedures use premiumProcedure (Premium gate)

ai.reports.generateTermReports({ classId, termId })
  // DEAN | HEADTEACHER
  // Queues Inngest job per student — returns jobId for polling
  
ai.reports.list({ entityType, entityId, termId? })
  // DEAN | HEADTEACHER | CLASS_TEACHER (own class)
  // Returns AIReport rows with status

ai.reports.updateDraft({ reportId, editedOutput })
  // DEAN | CLASS_TEACHER (review step)
  
ai.reports.approve({ reportId })
  // HEADTEACHER only → status = APPROVED → fires notification to parent

ai.reports.publish({ reportId })
  // HEADTEACHER → status = PUBLISHED → parent can view

ai.insights.getActive({ entityType?, severity? })
  // HEADTEACHER | DEAN | ACCOUNTANT | CLASS_TEACHER (own class insights)
  // Returns unacknowledged insights

ai.insights.acknowledge({ insightId })
  // Same roles as above

ai.feedback.request({ submissionId })
  // CLASS_TEACHER | SUBJECT_TEACHER (own assignment)
  // Queues Inngest job — returns immediate "generating" state

ai.feedback.getResult({ submissionId })
  // Poll endpoint — returns { status: 'pending'|'ready', options: [...] }
  // Expires 24h after generation

ai.timetable.requestReview({ timetableId })
  // DEAN_ACADEMICS only
  // Queues Inngest job — returns jobId

ai.timetable.getReviewResult({ jobId })
  // Returns { status, issues: [{ type, description, suggestion }] }

ai.lessonPlan.generateDraft({ subjectId, classId, topic, durationMinutes, objectives })
  // CLASS_TEACHER | SUBJECT_TEACHER
  // Queues Inngest job — returns lessonPlanId (pre-created as DRAFT)

ai.financial.generateNarrative({ reportId })
  // ACCOUNTANT | HEADTEACHER
  // Queues Inngest job — returns AIReport id

ai.usage.getStats({ from?, to? })
  // HEADTEACHER | IT_SUPPORT
```

---

## Inngest Jobs

### `generateTermReports`
```typescript
// inngest/functions/ai-term-reports.ts
export const generateTermReports = inngest.createFunction(
  {
    id: 'ai-generate-term-reports',
    concurrency: { limit: 5 },  // max 5 Gemini calls at once
    retries: 2,
  },
  { event: 'ai/term-reports.requested' },
  async ({ event, step }) => {
    const { schoolId, classId, termId, userId } = event.data

    // Get all students in class
    const students = await step.run('get-students', () =>
      prisma.student.findMany({ where: { schoolId, classId } })
    )

    // Generate report per student (fanned out as separate steps)
    await Promise.all(students.map(student =>
      step.run(`generate-${student.id}`, async () => {
        const data = await fetchStudentAcademicData(student.id, termId)
        const prompt = buildTermReportPrompt(student, data)
        
        const result = await callGemini(prompt, { maxOutputTokens: 800, temperature: 0.6 })
        
        await prisma.aIReport.create({
          data: {
            schoolId, reportType: 'TERM_REPORT', entityId: student.id,
            entityType: 'STUDENT', termId, prompt, rawOutput: result.text,
            tokensUsed: result.totalTokens, status: 'GENERATED',
          }
        })
        
        await logAIUsage({ schoolId, userId, feature: 'TERM_REPORT', ...result })
      })
    ))

    // Notify Dean that batch is complete
    await step.run('notify-dean', () =>
      createNotification(schoolId, userId, 'REPORT_READY',
        `Term reports generated for ${students.length} students`)
    )
  }
)
```

### `detectAnomalies`
```typescript
// inngest/functions/ai-anomaly-detection.ts
// Cron: "0 21 * * 1-5" (weekday evenings KE time)
// For each Premium school:
//   Run 4 detectors (attendance, fee, score, procurement)
//   Upsert AIInsight (@@unique prevents duplicates)
//   Create Notification for relevant staff
```

### `generateFeedbackSuggestions`
```typescript
// inngest/functions/ai-feedback.ts
// Trigger: ai/feedback.requested
// Fetch assignment + submission data
// Build prompt requesting 3 distinct options (JSON array)
// Parse JSON response carefully (try/catch around JSON.parse)
// Store in temporary cache key (Redis or DB temp table)
// 24h TTL
```

---

## Prompt Templates

### Term Report
```typescript
export function buildTermReportPrompt(student: Student, data: StudentAcademicData): string {
  return `You are writing an end-of-term academic report for ${student.firstName} ${student.lastName}, 
  studying in ${data.className} at ${data.schoolName}.

Academic summary this term:
- Overall attendance: ${data.attendancePercent}%
- Assignments submitted on time: ${data.onTimeSubmissionRate}%
- Subject performances: ${data.subjects.map(s => `${s.name}: ${s.grade}`).join(', ')}
- Teacher comments: ${data.teacherComments.join('; ')}

Write a warm, professional, constructive report (200-400 words).
Rules:
1. Do NOT mention specific numerical scores — use qualitative language (excellent, good, needs improvement)
2. Start with strengths
3. Include 1-2 specific areas for growth with actionable suggestions
4. End with an encouraging, forward-looking statement
5. Write in third person (refer to student by first name)
6. Output ONLY the report text — no headers, no preamble`
}
```

### Feedback Suggestions
```typescript
export function buildFeedbackPrompt(assignment: Assignment, score: number, maxMarks: number): string {
  const pct = Math.round((score / maxMarks) * 100)
  return `An assignment titled "${assignment.title}" was marked ${score}/${maxMarks} (${pct}%).

Write exactly 3 feedback options for the teacher to choose from.
Each option should be 2-3 sentences. Vary the tone:
- Option 1: Encouraging and supportive
- Option 2: Direct and specific  
- Option 3: Detailed with next steps

Output ONLY valid JSON (no preamble, no markdown):
[{"label":"Encouraging","text":"..."},{"label":"Direct","text":"..."},{"label":"Detailed","text":"..."}]`
}
```

---

## AI Banner Component

```tsx
// components/ai/AIBanner.tsx
export function AIBanner() {
  return (
    <div className="flex items-center gap-2 rounded-md bg-violet-50 border border-violet-200 px-3 py-2 text-sm text-violet-700">
      <SparklesIcon className="h-4 w-4 shrink-0" />
      <span>AI-generated — please review before publishing</span>
    </div>
  )
}
// Used on every page that displays AI-generated content
```

---

## Environment Variables (Sprint 007)

```env
GOOGLE_GENERATIVE_AI_API_KEY=   # Gemini API key
```
