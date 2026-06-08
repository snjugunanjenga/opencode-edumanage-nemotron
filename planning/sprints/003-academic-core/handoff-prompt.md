# Builder Handoff — Sprint 003: Academic Core

**Instructions:** Paste everything below this line into Claude Code.
Point Claude Code at the project root first. Sprint 002 must be fully passing.

---

You are a Builder agent implementing Sprint 003 of EduManage.
Sprint 002 (auth, multi-tenancy, 16 roles, onboarding wizard) is already complete.

## Mandatory pre-read (in order, before any code):

1. `AGENTS.md`
2. `CONTEXT.md`
3. `planning/DECISIONS.md`
4. `planning/DOMAIN.md`
5. `planning/RISKS.md`
6. `planning/sprints/003-academic-core/requirements.md`
7. `planning/sprints/003-academic-core/blueprint.md`
8. `planning/sprints/003-academic-core/acceptance.md`

## Agent routing:

- **New Prisma models / tRPC routers / server actions / API routes** → use `.claude/agents/nextjs-expert.md`
- **React components: charts, calendar, quiz UI, forms** → use `.claude/agents/react-frontend.md`
- **Any Clerk role checks or middleware changes** → use `.claude/agents/clerk-auth.md`
- **Inngest jobs, CI changes, environment variables** → use `.claude/agents/devops.md`

## Summarise before coding

Before writing any code, output:
1. All new files you will create (full path)
2. All existing files you will modify (full path + what changes)
3. New Prisma migration names
4. New Inngest jobs
5. New environment variables needed (none expected for this sprint)
6. Any `// ARCH-QUESTION:` items

Wait for operator approval. Then build.

---

## Sprint Goal

Deliver timetabling, attendance, assignments, quizzes, exams, performance analytics, unified calendar, lesson plans, and syllabus management. After this sprint, the platform is demonstrable to pilot schools.

---

## Implementation Sequence

### Phase 1: Database Schema

1. Add all Sprint 003 Prisma models to `prisma/schema.prisma` (full schema in blueprint.md)
2. Run: `prisma migrate dev --name sprint_003_academic_core`
3. Apply manual RLS policies for all new tables:

```sql
-- Add to prisma/migrations/manual/002_rls_sprint003.sql
ALTER TABLE subjects ENABLE ROW LEVEL SECURITY;
ALTER TABLE rooms ENABLE ROW LEVEL SECURITY;
ALTER TABLE timetables ENABLE ROW LEVEL SECURITY;
ALTER TABLE timetable_slots ENABLE ROW LEVEL SECURITY;
ALTER TABLE attendance_records ENABLE ROW LEVEL SECURITY;
ALTER TABLE assignments ENABLE ROW LEVEL SECURITY;
ALTER TABLE assignment_submissions ENABLE ROW LEVEL SECURITY;
ALTER TABLE quizzes ENABLE ROW LEVEL SECURITY;
ALTER TABLE quiz_questions ENABLE ROW LEVEL SECURITY;
ALTER TABLE quiz_attempts ENABLE ROW LEVEL SECURITY;
ALTER TABLE quiz_answers ENABLE ROW LEVEL SECURITY;
ALTER TABLE exams ENABLE ROW LEVEL SECURITY;
ALTER TABLE exam_results ENABLE ROW LEVEL SECURITY;
ALTER TABLE analytics_summaries ENABLE ROW LEVEL SECURITY;
ALTER TABLE calendar_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE calendar_event_attendees ENABLE ROW LEVEL SECURITY;
ALTER TABLE lesson_plans ENABLE ROW LEVEL SECURITY;
ALTER TABLE syllabuses ENABLE ROW LEVEL SECURITY;
ALTER TABLE syllabus_topics ENABLE ROW LEVEL SECURITY;
ALTER TABLE syllabus_progress ENABLE ROW LEVEL SECURITY;

-- Apply tenant_isolation policy to all (same pattern as Sprint 002)
-- Use school_id column for all except syllabus_topics (joins via syllabus)
```

4. Create DB view for calendar query performance:

```sql
-- prisma/migrations/manual/003_calendar_view.sql
CREATE OR REPLACE VIEW user_visible_events AS
SELECT e.*, 
  CASE 
    WHEN e.visibility_scope = 'SCHOOL' THEN true
    WHEN e.visibility_scope = 'PERSONAL' AND e.created_by_user_id = current_setting('app.current_user_id', true)::uuid THEN true
    WHEN e.visibility_scope = 'CLASS' THEN true  -- filtered at query time
    WHEN e.visibility_scope = 'CLUB' THEN true   -- filtered at query time
    ELSE false
  END as is_visible
FROM calendar_events e
WHERE e.school_id = current_setting('app.current_school_id', true)::uuid;
```

### Phase 2: tRPC Routers

5. Create `server/routers/academic.ts` with sub-routers:
   - `timetable.*` (create, addSlot, detectConflicts, publish, getByClass)
   - `attendance.*` (bulkMark, getByClass, getByStudent, getSummary)
   - `assignment.*` (create, publish, submit, mark, listForClass, listForStudent)
   - `quiz.*` (create, addQuestion, publish, startAttempt, saveAnswer, submitAttempt, gradeShortAnswer, releaseResults, getResults)
   - `exam.*` (create, uploadPaper, enterResults, approve, release, getResults)
   - `lessonPlan.*` (create, publish, list)
   - `syllabus.*` (upload, addTopics, markTaught, getByClass)

6. Create `server/routers/analytics.ts`:
   - `getStudentAnalytics({ studentId, termId })`
   - `getClassAnalytics({ classId, termId })`
   - `getSchoolAnalytics({ termId })`
   - `getTeacherAnalytics({ teacherId, termId })`

7. Create `server/routers/calendar.ts`:
   - `getMyEvents({ startDate, endDate })`
   - `createPersonalEvent(...)`
   - `createSchoolEvent(...)` (HEADTEACHER, DEAN only)
   - `createClassEvent(...)` (CLASS_TEACHER, DEAN only)
   - `exportIcs({ startDate, endDate })`

8. Add all new routers to `server/routers/index.ts` (appRouter)

### Phase 3: Inngest Jobs

9. Create `inngest/functions/quiz-autograde.ts`:
   - Trigger: `quiz/attempt.submitted`
   - Grades MCQ (exact match) and True/False answers
   - Computes `totalScore`, `percentageScore`, `isPassed`
   - Updates `QuizAttempt` and `QuizAnswer` records
   - Fires `analytics/recompute` event

10. Create `inngest/functions/analytics-recompute.ts`:
    - Trigger: `analytics/recompute`
    - Data: `{ schoolId, studentId, classId, subjectId, termId }`
    - Queries ExamResult + QuizAttempt + AttendanceRecord + AssignmentSubmission
    - Computes metrics and upserts `AnalyticsSummary`

11. Create `inngest/functions/attendance-alert.ts`:
    - Trigger: `inngest/scheduled` — daily cron `0 20 * * 1-5` (8pm KE weekdays)
    - Finds students with ≥3 consecutive ABSENT days
    - Creates in-app Notification records for Class Teacher + Dean + Parent

12. Create `inngest/functions/timetable-create-events.ts`:
    - Trigger: `timetable/published`
    - Creates `CalendarEvent` rows for each slot for each affected student + teacher

13. Create `inngest/functions/assignment-create-event.ts`:
    - Trigger: `assignment/published`
    - Creates `CalendarEvent` (due date) for all students in the class

14. Create `inngest/functions/quiz-create-event.ts`:
    - Trigger: `quiz/published`
    - Creates `CalendarEvent` (quiz time) for all students in the class

### Phase 4: Pages & Components

15. **Calendar page** (`app/(school)/[schoolSlug]/calendar/page.tsx`):
    - Server component fetches events for next 90 days
    - Passes to `<UnifiedCalendar>` client component

16. **UnifiedCalendar component** (`components/calendar/UnifiedCalendar.tsx`):
    - `'use client'`
    - Uses `react-big-calendar` with `date-fns` localizer
    - `<Activity>` preserves view state between tab switches
    - Event type colour mapping (see requirements.md FR-007)
    - Filter chips by event type
    - Click → shadcn Sheet with event details
    - "+" button → quick-create personal event (shadcn Dialog + form)
    - "Export .ics" button → calls `calendar.exportIcs` tRPC, triggers download

17. **Analytics pages** (`app/(school)/[schoolSlug]/analytics/*`):
    - Server component reads role from Clerk, renders correct analytics variant
    - Pass pre-computed `AnalyticsSummary` data as props to chart components

18. **Analytics chart components** (all `'use client'` + Recharts):
    - `components/analytics/StudentScoreTrend.tsx` — LineChart, score over time
    - `components/analytics/SubjectRadar.tsx` — RadarChart, scores by subject
    - `components/analytics/AttendanceBar.tsx` — BarChart, monthly attendance %
    - `components/analytics/AssignmentDonut.tsx` — PieChart, submission status
    - `components/analytics/ClassPerformanceTable.tsx` — sortable table, top/bottom 10% highlighted
    - `components/analytics/SchoolAttendanceHeatmap.tsx` — calendar heatmap

19. **Quiz Taking UI** (`components/academic/quizzes/QuizAttemptUI.tsx`):
    - `'use client'`
    - Server-derived start time + duration passed as props
    - Client countdown timer (display only — server enforces)
    - Question navigator sidebar with flagging
    - Answer state managed with `useReducer`
    - Auto-save answer on change (debounced 1s) via tRPC mutation
    - "Submit Quiz" → confirmation dialog → `quiz.submitAttempt` call

20. **Attendance Register** (`components/academic/attendance/AttendanceRegister.tsx`):
    - `'use client'`
    - "Mark All Present" button with `useOptimistic`
    - Individual student status toggle (Present/Absent/Late/Excused)
    - Bulk save on "Submit Register" click
    - Disabled after 24 hours

21. **All remaining pages** — timetable designer, assignment create/submit/mark, quiz builder, exam results entry, lesson plan editor, syllabus manager

### Phase 5: Tests

22. Integration tests:
    - Quiz auto-grade logic (unit test — pure function)
    - Attendance 7-day lockout (tRPC error test)
    - RLS on quiz attempts (cross-tenant query returns 0)
    - Analytics recompute correctness (seed data → trigger → check summaries)

23. E2E tests (Playwright):
    - Full assignment lifecycle: create → publish → student submits → teacher marks → student sees result
    - Full quiz lifecycle: create → publish → student attempts → auto-graded → results released → student views
    - Calendar shows correct events for Teacher + Student roles

---

## Critical Implementation Notes

### Quiz Timer Security
```typescript
// In quiz.submitAttempt tRPC procedure
const attempt = await ctx.prisma.quizAttempt.findUnique({ where: { id: input.attemptId } })
const quiz = await ctx.prisma.quiz.findUnique({ where: { id: attempt.quizId } })
const allowedUntil = new Date(attempt.startedAt.getTime() + (quiz.durationMinutes + 5) * 60000)

if (new Date() > allowedUntil) {
  // Still process but flag as auto-submitted
  await ctx.prisma.quizAttempt.update({
    where: { id: attempt.id },
    data: { isAutoSubmitted: true, submittedAt: new Date() }
  })
}
// Then proceed with grading regardless
```

### Analytics Query (NEVER query raw tables from charts)
```typescript
// server/routers/analytics.ts
async getStudentAnalytics({ studentId, termId }) {
  const summaries = await ctx.prisma.analyticsSummary.findMany({
    where: { 
      schoolId: ctx.schoolId,
      entityType: 'STUDENT', 
      entityId: studentId,
      termId 
    }
  })
  // Transform into chart data shapes
  return transformToChartData(summaries)
}
```

### Calendar .ics Export
```typescript
// Use 'ical-generator' npm package
import ical from 'ical-generator'

const cal = ical({ name: `${school.name} Calendar` })
events.forEach(event => {
  cal.createEvent({
    start: event.startAt,
    end: event.endAt,
    summary: event.title,
    description: event.description,
  })
})
return cal.toString()  // RFC 5545 format string
```

### Packages to Install
```bash
npm install react-big-calendar date-fns ical-generator
npm install -D @types/react-big-calendar
```

---

## Definition of Done

All acceptance criteria in `acceptance.md` pass.
No unresolved `// ARCH-QUESTION:` comments.
RLS policies applied for all 20 new tables.
All 6 Inngest jobs implemented and tested locally with `inngest-cli dev`.
Calendar renders correctly for Teacher, Student, and Parent roles.
Analytics page renders all charts from `AnalyticsSummary` (not raw tables).
Quiz timer enforced server-side — overtime submit still accepted but flagged.
`CONTEXT.md` Sprint History updated: Sprint 003 marked complete.
`STATE.md` updated: Sprint 004 is next.
