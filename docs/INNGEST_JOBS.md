# INNGEST_JOBS.md — EduManage Async Job Reference

**Last updated:** 2026-05-29  
**Runtime:** Inngest hosted, deployed via Vercel  
**Dev tool:** `npx inngest-cli@latest dev` (starts local Inngest server at http://localhost:8288)

---

## Setup

```typescript
// inngest/client.ts
import { Inngest } from 'inngest'
export const inngest = new Inngest({ id: 'edumanage' })
```

```typescript
// app/api/inngest/route.ts
import { serve } from 'inngest/next'
import { inngest } from '@/inngest/client'
import { allFunctions } from '@/inngest/functions'

export const { GET, POST, PUT } = serve({ client: inngest, functions: allFunctions })
```

---

## All Jobs — Master List

### Sprint 003 — Academic Core

| Function ID | Trigger | Schedule | Purpose |
|---|---|---|---|
| `quiz-autograde` | Event: `quiz/attempt.submitted` | On-demand | Auto-grade MCQ + True/False answers, compute score |
| `analytics-recompute` | Event: `analytics/recompute` | On-demand | Recompute AnalyticsSummary for student/class/subject |
| `attendance-alert-absence` | Cron: `0 20 * * 1-5` | Nightly 8pm KE (Mon–Fri) | Alert Class Teacher + Dean + Parent if ≥3 consecutive absences |
| `timetable-create-events` | Event: `timetable/published` | On-demand | Create CalendarEvent rows for all timetable slots |
| `assignment-create-event` | Event: `assignment/published` | On-demand | Create CalendarEvent (due date) for class members |
| `quiz-create-event` | Event: `quiz/published` | On-demand | Create CalendarEvent (quiz time) for class members |

---

### Sprint 004 — Financial

| Function ID | Trigger | Schedule | Purpose |
|---|---|---|---|
| `process-mpesa-payment` | Event: `mpesa/callback.received` | On-demand | Idempotent fee payment processing + receipt email |
| `generate-payslips` | Event: `payroll/run.approved` | On-demand | Generate PDF payslip per staff member + email |
| `fee-reminder` | Cron: `0 8 * * 1` | Every Monday 8am KE | Email + in-app reminder for outstanding fee balances |
| `subscription-expiry-check` | Cron: `0 6 * * *` | Daily 6am KE | TRIAL→PAST_DUE after 1 day, PAST_DUE→SUSPENDED after 14 days |
| `bank-reconcile` | Event: `bank/statement.uploaded` | On-demand | Parse bank CSV, auto-match to FeePayments |
| `generate-financial-report` | Event: `reports/generate.requested` | On-demand | Generate PDF/Excel report → S3 → notify user |
| `stripe-webhook-handler` | Event: `stripe/webhook.received` | On-demand | Update SchoolSubscription from Stripe events |

---

### Sprint 005 — Communications & Procurement

| Function ID | Trigger | Schedule | Purpose |
|---|---|---|---|
| `send-message-notification` | Event: `messaging/message.sent` | On-demand | In-app + conditional email notification per thread member |
| `send-notification` | Event: `notifications/send` | On-demand | Generic notification creation + conditional email |
| `procurement-next-approver` | Event: `procurement/step.completed` | On-demand | Determine next approver in chain + notify them |
| `leave-approved` | Event: `leave/request.approved` | On-demand | Update timetable slots, create CalendarEvent for substitute, notify substitute |

---

### Sprint 006 — Library

| Function ID | Trigger | Schedule | Purpose |
|---|---|---|---|
| `recalculate-storage-usage` | Cron: `0 2 * * *` | Daily 2am KE | Sum S3 object sizes per school, update StorageUsage, alert at 90%/100% |
| `library-resource-approved` | Event: `library/resource.approved` | On-demand | Notify uploader that resource is approved and live |
| `increment-download-count` | Event: `library/resource.downloaded` | On-demand | Async increment of LibraryResource.downloadCount |

---

### Sprint 007 — AI Automation (Premium only)

| Function ID | Trigger | Schedule | Concurrency | Purpose |
|---|---|---|---|---|
| `ai-generate-term-reports` | Event: `ai/term-reports.requested` | On-demand | 5 | Batch Gemini term report generation per student |
| `ai-detect-anomalies` | Cron: `0 21 * * 1-5` | Nightly 9pm KE (Mon–Fri) | 10 | Run 4 anomaly detectors across all Premium schools |
| `ai-feedback-suggestions` | Event: `ai/feedback.requested` | On-demand | 10 | Generate 3 feedback options for assignment marking |
| `ai-timetable-review` | Event: `ai/timetable-review.requested` | On-demand | 5 | Gemini timetable analysis — soft issues only |
| `ai-lesson-plan-draft` | Event: `ai/lesson-plan.requested` | On-demand | 10 | Gemini lesson plan generation → LessonPlan DRAFT |
| `ai-financial-narrative` | Event: `ai/financial-narrative.requested` | On-demand | 5 | Gemini financial report summary |
| `ai-cleanup-expired-feedback` | Cron: `0 3 * * *` | Daily 3am KE | 1 | Delete AIFeedbackSuggestion rows where expiresAt < now() |

---

### Sprint 008 — Departments

| Function ID | Trigger | Schedule | Purpose |
|---|---|---|---|
| `vehicle-insurance-expiry` | Cron: `0 8 * * 1` | Weekly Monday 8am KE | Alert Transport Manager + HT for vehicles expiring ≤30 days |
| `contract-expiry-alert` | Cron: `0 8 1 * *` | 1st of each month 8am KE | Alert HR Manager for contracts expiring ≤60 days |
| `kitchen-reorder-check` | Cron: `0 7 * * *` | Daily 7am KE | Create DRAFT procurement for items below reorder level (idempotent) |
| `appraisal-cycle-start` | Event: `hr/appraisal-cycle.started` | On-demand (manual trigger) | Create PerformanceAppraisal DRAFT rows for all active staff |
| `trip-approved-calendar` | Event: `transport/trip.approved` | On-demand | Create CalendarEvent (TRIP) for all manifest students + chaperones |

---

## Total Job Count: 29

---

## Retry Configuration

```typescript
// Default retry config (apply to all jobs unless overridden)
{
  retries: 3,
  // Inngest uses exponential backoff: 1s, 2s, 4s
}

// Critical financial jobs — higher retries
// process-mpesa-payment, generate-payslips
{
  retries: 5,
}

// AI jobs — fewer retries (expensive tokens)
// ai-generate-term-reports, ai-detect-anomalies
{
  retries: 2,
}
```

---

## Idempotency Patterns

### Mpesa Payment (process-mpesa-payment)
```typescript
// Check receipt exists before processing
const existing = await prisma.feePayment.findUnique({
  where: { mpesaReceiptNo: event.data.receiptNo }
})
if (existing) return { skipped: true, reason: 'duplicate_receipt' }
// Proceed with payment creation
```

### Anomaly Detection (ai-detect-anomalies)
```typescript
// @@unique([schoolId, insightType, entityId, termId]) on AIInsight
// Use upsert — DB constraint prevents duplicates even on race conditions
await prisma.aIInsight.upsert({
  where: { schoolId_insightType_entityId_termId: { schoolId, insightType, entityId, termId } },
  create: { ...insight },
  update: { detectedAt: new Date() } // only update timestamp on re-detection
})
```

### Kitchen Reorder (kitchen-reorder-check)
```typescript
// Check for existing DRAFT/PENDING request for same item this week
const weekStart = startOfWeek(new Date())
const existing = await prisma.procurementRequest.findFirst({
  where: {
    schoolId,
    department: 'KITCHEN',
    status: { in: ['DRAFT', 'PENDING_HOD', 'PENDING_PROCUREMENT', 'PENDING_ACCOUNTANT', 'PENDING_HEAD'] },
    createdAt: { gte: weekStart }
  }
})
if (existing) return { skipped: true }
```

---

## Event Types Reference

```typescript
// All Inngest event names used in EduManage
type EduManageEvents = {
  // Academic
  'quiz/attempt.submitted':        { attemptId: string; quizId: string; schoolId: string }
  'analytics/recompute':           { schoolId: string; studentId?: string; classId?: string; subjectId?: string; termId: string }
  'timetable/published':           { timetableId: string; schoolId: string }
  'assignment/published':          { assignmentId: string; schoolId: string; classId: string }
  'quiz/published':                { quizId: string; schoolId: string; classId: string }

  // Financial
  'mpesa/callback.received':       { checkoutRequestId: string; resultCode: number; resultDesc: string; callbackMetadata: unknown }
  'payroll/run.approved':          { salaryRunId: string; schoolId: string }
  'bank/statement.uploaded':       { statementId: string; schoolId: string }
  'reports/generate.requested':    { reportType: string; parameters: unknown; schoolId: string; userId: string }
  'stripe/webhook.received':       { type: string; data: unknown }

  // Communications
  'messaging/message.sent':        { messageId: string; threadId: string; schoolId: string; senderId: string }
  'notifications/send':            { schoolId: string; userId: string; type: string; title: string; body: string; metadata?: unknown }
  'procurement/step.completed':    { requestId: string; schoolId: string; newStatus: string; nextApproverId?: string }
  'leave/request.approved':        { requestId: string; schoolId: string; substituteTeacherId?: string }

  // Library
  'library/resource.approved':     { resourceId: string; schoolId: string; uploadedByUserId: string }
  'library/resource.downloaded':   { resourceId: string; schoolId: string }

  // AI
  'ai/term-reports.requested':     { schoolId: string; classId: string; termId: string; userId: string }
  'ai/feedback.requested':         { submissionId: string; assignmentId: string; schoolId: string; userId: string; score: number; maxMarks: number }
  'ai/timetable-review.requested': { timetableId: string; schoolId: string; userId: string }
  'ai/lesson-plan.requested':      { lessonPlanId: string; schoolId: string; userId: string; subjectId: string; topic: string; objectives: string[] }
  'ai/financial-narrative.requested': { reportId: string; schoolId: string; userId: string }
  'hr/appraisal-cycle.started':    { schoolId: string; cycleYear: number; userId: string }

  // Transport
  'transport/trip.approved':       { tripId: string; schoolId: string }
}
```

---

## Monitoring Alerts

Configure in Inngest dashboard → Project → Alerts:

| Alert | Condition | Notify |
|---|---|---|
| Payment job failure | `process-mpesa-payment` fails after all retries | Slack #payments-alerts |
| Payslip job failure | `generate-payslips` fails | Slack #hr-alerts |
| AI batch failure | `ai-generate-term-reports` fails | Slack #ai-alerts |
| High queue depth | Any queue >100 pending | Slack #platform-alerts |
| Anomaly detection late | `ai-detect-anomalies` not started by 22:00 | PagerDuty |

---

## Local Development

```bash
# Terminal 1: Next.js dev server
npm run dev

# Terminal 2: Inngest dev server
npx inngest-cli@latest dev

# Terminal 3: Trigger test events
curl -X POST http://localhost:8288/e/local \
  -H "Content-Type: application/json" \
  -d '{"name":"quiz/attempt.submitted","data":{"attemptId":"xxx","quizId":"yyy","schoolId":"zzz"}}'
```

Inngest UI available at `http://localhost:8288` — shows all functions, events, and run history.
