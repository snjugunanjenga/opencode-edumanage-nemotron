# Blueprint — Sprint 003: Academic Core

**Project:** EduManage  
**Sprint:** 003  
**Status:** Approved  
**Last updated:** 2026-05-29  
**Prerequisite:** Sprint 002 acceptance criteria all passing

---

## Architecture Additions

### New Prisma Models (complete schema additions)

```prisma
// ─── Academic Entities ───────────────────────────────────────────

model Subject {
  id           String @id @default(uuid())
  schoolId     String
  name         String
  code         String         // e.g. "ENG", "MAT", "SCI"
  curriculumType CurriculumType
  school       School @relation(fields: [schoolId], references: [id])
  @@unique([schoolId, code])
  @@index([schoolId])
}

model Room {
  id       String @id @default(uuid())
  schoolId String
  name     String
  capacity Int
  building String?
  @@index([schoolId])
}

model Timetable {
  id          String @id @default(uuid())
  schoolId    String
  termId      String
  classId     String
  name        String
  isPublished Boolean @default(false)
  publishedAt DateTime?
  slots       TimetableSlot[]
  @@index([schoolId])
}

model TimetableSlot {
  id           String    @id @default(uuid())
  timetableId  String
  schoolId     String
  subjectId    String
  teacherId    String    // User.id
  roomId       String?
  dayOfWeek    Int       // 1=Mon … 5=Fri
  periodNumber Int
  startTime    String    // "08:00"
  endTime      String    // "08:40"
  timetable    Timetable @relation(fields: [timetableId], references: [id])
  @@index([schoolId, timetableId])
}

// ─── Attendance ──────────────────────────────────────────────────

model AttendanceRecord {
  id              String           @id @default(uuid())
  schoolId        String
  classId         String
  studentId       String
  date            DateTime         @db.Date
  status          AttendanceStatus // PRESENT | ABSENT | LATE | EXCUSED
  notes           String?
  markedByUserId  String
  createdAt       DateTime         @default(now())
  @@unique([schoolId, classId, studentId, date])
  @@index([schoolId, classId, date])
  @@index([schoolId, studentId])
}

enum AttendanceStatus { PRESENT ABSENT LATE EXCUSED }

// ─── Assignments ─────────────────────────────────────────────────

model Assignment {
  id                  String             @id @default(uuid())
  schoolId            String
  subjectId           String
  classId             String
  teacherId           String
  title               String
  description         String
  dueDate             DateTime
  maxMarks            Int
  allowLateSubmission Boolean            @default(true)
  latePenaltyPercent  Int                @default(0)
  attachmentS3Key     String?
  status              AssignmentStatus   @default(DRAFT)  // DRAFT | PUBLISHED | CLOSED
  createdAt           DateTime           @default(now())
  submissions         AssignmentSubmission[]
  @@index([schoolId, classId])
  @@index([schoolId, teacherId])
}

model AssignmentSubmission {
  id               String             @id @default(uuid())
  assignmentId     String
  studentId        String
  schoolId         String
  submittedAt      DateTime?
  isLate           Boolean            @default(false)
  textResponse     String?
  attachmentS3Key  String?
  score            Int?
  feedback         String?
  markedAt         DateTime?
  markedByUserId   String?
  status           SubmissionStatus   @default(PENDING)
  assignment       Assignment         @relation(fields: [assignmentId], references: [id])
  @@unique([assignmentId, studentId])
  @@index([schoolId, studentId])
}

enum AssignmentStatus  { DRAFT PUBLISHED CLOSED }
enum SubmissionStatus  { PENDING SUBMITTED MARKED RETURNED }

// ─── Quizzes ─────────────────────────────────────────────────────

model Quiz {
  id               String        @id @default(uuid())
  schoolId         String
  subjectId        String
  classId          String
  teacherId        String
  title            String
  description      String?
  scheduledAt      DateTime
  durationMinutes  Int
  attemptsAllowed  Int           @default(1)
  passMark         Int           @default(50)  // percentage
  isPublished      Boolean       @default(false)
  resultsReleased  Boolean       @default(false)
  totalMarks       Int           // computed on publish
  createdAt        DateTime      @default(now())
  questions        QuizQuestion[]
  attempts         QuizAttempt[]
  @@index([schoolId, classId])
}

model QuizQuestion {
  id                 String       @id @default(uuid())
  quizId             String
  schoolId           String
  questionText       String
  questionType       QuestionType // MCQ | TRUE_FALSE | SHORT_ANSWER
  options            Json?        // String[] for MCQ
  correctOptionIndex Int?         // 0-indexed, MCQ only
  correctAnswer      Boolean?     // TRUE_FALSE only
  marks              Int          @default(1)
  orderIndex         Int
  explanation        String?      // shown after results released
  quiz               Quiz         @relation(fields: [quizId], references: [id])
  answers            QuizAnswer[]
  @@index([quizId])
}

enum QuestionType { MCQ TRUE_FALSE SHORT_ANSWER }

model QuizAttempt {
  id                String         @id @default(uuid())
  quizId            String
  studentId         String
  schoolId          String
  startedAt         DateTime       @default(now())
  submittedAt       DateTime?
  isAutoSubmitted   Boolean        @default(false)
  totalScore        Int?
  percentageScore   Decimal?       @db.Decimal(5,2)
  isPassed          Boolean?
  status            AttemptStatus  @default(IN_PROGRESS)
  quiz              Quiz           @relation(fields: [quizId], references: [id])
  answers           QuizAnswer[]
  @@unique([quizId, studentId])   // one attempt per student (until attemptsAllowed >1)
  @@index([schoolId, studentId])
}

model QuizAnswer {
  id                  String       @id @default(uuid())
  attemptId           String
  questionId          String
  schoolId            String
  selectedOptionIndex Int?
  selectedBoolean     Boolean?
  textAnswer          String?
  isCorrect           Boolean?
  marksAwarded        Int?
  teacherFeedback     String?
  attempt             QuizAttempt  @relation(fields: [attemptId], references: [id])
  question            QuizQuestion @relation(fields: [questionId], references: [id])
  @@unique([attemptId, questionId])
}

enum AttemptStatus { IN_PROGRESS SUBMITTED GRADED }

// ─── Exams ───────────────────────────────────────────────────────

model Exam {
  id              String       @id @default(uuid())
  schoolId        String
  classId         String
  subjectId       String
  teacherId       String
  title           String
  examType        ExamType     // CAT | MID_TERM | END_TERM | MOCK | PRACTICAL
  examDate        DateTime     @db.Date
  maxMarks        Int
  passMark        Int          @default(50)
  paperS3Key      String?      // encrypted
  paperReleasedAt DateTime?
  status          ExamStatus   @default(DRAFT)
  approvedByUserId String?
  approvedAt      DateTime?
  releasedAt      DateTime?
  createdAt       DateTime     @default(now())
  results         ExamResult[]
  @@index([schoolId, classId])
}

model ExamResult {
  id              String  @id @default(uuid())
  examId          String
  studentId       String
  schoolId        String
  rawScore        Int?
  grade           String?  // "Exceeding" | "A*" etc — string for flexibility
  isAbsent        Boolean  @default(false)
  teacherComment  String?
  exam            Exam     @relation(fields: [examId], references: [id])
  @@unique([examId, studentId])
  @@index([schoolId, studentId])
}

enum ExamType    { CAT MID_TERM END_TERM MOCK PRACTICAL }
enum ExamStatus  { DRAFT SUBMITTED APPROVED RELEASED }

// ─── Analytics ───────────────────────────────────────────────────

model AnalyticsSummary {
  id          String              @id @default(uuid())
  schoolId    String
  entityType  AnalyticsEntityType // STUDENT | CLASS | SUBJECT | SCHOOL
  entityId    String              // studentId | classId | subjectId | schoolId
  termId      String
  metric      String              // "avg_score" | "attendance_pct" | "pass_rate" etc
  value       Decimal             @db.Decimal(8, 2)
  computedAt  DateTime            @default(now())
  @@unique([schoolId, entityType, entityId, termId, metric])
  @@index([schoolId, entityType, entityId])
}

enum AnalyticsEntityType { STUDENT CLASS SUBJECT SCHOOL }

// ─── Calendar ────────────────────────────────────────────────────

model CalendarEvent {
  id              String          @id @default(uuid())
  schoolId        String
  title           String
  description     String?
  startAt         DateTime
  endAt           DateTime
  isAllDay        Boolean         @default(false)
  eventType       CalendarEventType
  sourceId        String?         // FK to originating record
  sourceType      String?         // "Assignment" | "Quiz" | "Exam" | "TimetableSlot"
  createdByUserId String
  visibilityScope EventScope      // PERSONAL | CLASS | SCHOOL | CLUB
  scopeEntityId   String?         // classId | clubId
  colour          String?         // hex override
  createdAt       DateTime        @default(now())
  attendees       CalendarEventAttendee[]
  @@index([schoolId, startAt])
  @@index([schoolId, visibilityScope, scopeEntityId])
}

model CalendarEventAttendee {
  eventId   String
  userId    String
  schoolId  String
  status    AttendeeStatus @default(INVITED)
  event     CalendarEvent  @relation(fields: [eventId], references: [id])
  @@id([eventId, userId])
  @@index([schoolId, userId])
}

enum CalendarEventType {
  TIMETABLE ASSIGNMENT QUIZ EXAM SCHOOL CLASS CLUB TRIP LEAVE PERSONAL
}
enum EventScope     { PERSONAL CLASS SCHOOL CLUB }
enum AttendeeStatus { INVITED ACCEPTED DECLINED }

// ─── Lesson Plans & Syllabus ─────────────────────────────────────

model LessonPlan {
  id              String           @id @default(uuid())
  schoolId        String
  teacherId       String
  classId         String
  subjectId       String
  date            DateTime         @db.Date
  topic           String
  objectives      Json             // String[]
  activities      Json             // String[]
  resourcesNeeded String?
  status          LessonPlanStatus @default(DRAFT)
  docxS3Key       String?
  createdAt       DateTime         @default(now())
  @@index([schoolId, teacherId])
  @@index([schoolId, classId, subjectId])
}

enum LessonPlanStatus { DRAFT PUBLISHED }

model Syllabus {
  id             String           @id @default(uuid())
  schoolId       String
  subjectId      String
  classId        String
  termId         String
  fileS3Key      String
  uploadedByUserId String
  createdAt      DateTime         @default(now())
  topics         SyllabusTopic[]
  @@unique([schoolId, subjectId, classId, termId])
}

model SyllabusTopic {
  id          String             @id @default(uuid())
  syllabusId  String
  schoolId    String
  title       String
  description String?
  orderIndex  Int
  syllabus    Syllabus           @relation(fields: [syllabusId], references: [id])
  progress    SyllabusProgress[]
}

model SyllabusProgress {
  syllabusTopicId String
  classId         String
  teacherId       String
  schoolId        String
  taughtAt        DateTime?
  @@id([syllabusTopicId, classId, teacherId])
}
```

---

## API Contracts (tRPC Routers)

### `academic.timetable.*`
- `create({ termId, classId, name })` → DEAN only
- `addSlot({ timetableId, subjectId, teacherId, roomId, dayOfWeek, periodNumber, startTime, endTime })` → DEAN
- `detectConflicts({ timetableId })` → returns `ConflictResult[]`
- `publish({ timetableId })` → DEAN — triggers Inngest to create CalendarEvents
- `getByClass({ classId, termId })` → Teacher, Student, Parent (own class)

### `academic.attendance.*`
- `bulkMark({ classId, date, records: [{studentId, status, notes?}] })` → CLASS_TEACHER
- `getByClass({ classId, date })` → CLASS_TEACHER, DEAN, HEADTEACHER
- `getByStudent({ studentId, termId })` → STUDENT (own), PARENT (own child), CLASS_TEACHER, DEAN
- `getSummary({ classId, termId })` → DEAN, HEADTEACHER

### `academic.assignment.*`
- `create({ subjectId, classId, title, ... })` → CLASS_TEACHER, SUBJECT_TEACHER
- `publish({ assignmentId })` → assignment.teacherId only
- `submit({ assignmentId, textResponse?, attachmentS3Key? })` → STUDENT (own class)
- `mark({ submissionId, score, feedback })` → assignment.teacherId only
- `listForClass({ classId })` → Teacher (own), DEAN
- `listForStudent({ studentId? })` → STUDENT (own), PARENT (own child)

### `academic.quiz.*`
- `create({ subjectId, classId, ... })` → CLASS_TEACHER, SUBJECT_TEACHER
- `addQuestion({ quizId, questionText, questionType, ... })` → quiz.teacherId
- `publish({ quizId })` → quiz.teacherId — validates all questions have answers
- `startAttempt({ quizId })` → STUDENT — validates within time window
- `saveAnswer({ attemptId, questionId, ... })` → STUDENT (own attempt, within time)
- `submitAttempt({ attemptId })` → STUDENT — triggers Inngest auto-grade job
- `gradeShortAnswer({ answerId, marksAwarded, teacherFeedback })` → quiz.teacherId
- `releaseResults({ quizId })` → quiz.teacherId — requires all SHORT_ANSWER graded
- `getResults({ quizId })` → Teacher (all), STUDENT (own after release), DEAN

### `academic.exam.*`
- `create({ classId, subjectId, title, examType, examDate, maxMarks })` → SUBJECT_TEACHER
- `uploadPaper({ examId, s3Key })` → exam.teacherId — requires password confirmation
- `enterResults({ examId, results: [{studentId, rawScore, isAbsent, comment?}] })` → exam.teacherId
- `approve({ examId })` → DEAN, HEADTEACHER — triggers CalendarEvent + notification
- `release({ examId })` → DEAN, HEADTEACHER — makes results visible to students/parents
- `getResults({ examId })` → Teacher (all), STUDENT (own, after release), PARENT (own child, after release)

### `academic.calendar.*`
- `getMyEvents({ startDate, endDate })` → ALL roles — returns personalised events
- `createPersonalEvent({ title, startAt, endAt, ... })` → ALL roles
- `createSchoolEvent({ title, startAt, endAt, ... })` → HEADTEACHER, DEAN
- `createClassEvent({ classId, title, ... })` → CLASS_TEACHER, DEAN
- `exportIcs({ startDate, endDate })` → ALL roles — returns .ics file content

### `analytics.*`
- `getStudentAnalytics({ studentId, termId })` → STUDENT (own), PARENT (own child), CLASS_TEACHER, DEAN
- `getClassAnalytics({ classId, termId })` → CLASS_TEACHER, DEAN, HEADTEACHER
- `getSchoolAnalytics({ termId })` → DEAN, HEADTEACHER
- `getTeacherAnalytics({ teacherId, termId })` → SUBJECT_TEACHER (own), DEAN, HEADTEACHER

---

## Inngest Jobs (Sprint 003 additions)

| Job | Trigger | Action |
|---|---|---|
| `quiz.autoGrade` | QuizAttempt submitted | Grade MCQ/True-False answers, compute score, notify student |
| `exam.conflictCheck` | ExamResult entered | Detect duplicate entries |
| `analytics.recompute` | ExamResult approved / QuizAttempt graded / Attendance marked | Update AnalyticsSummary for affected student/class/subject |
| `attendance.alertAbsence` | Nightly at 20:00 KE | Find students with ≥3 consecutive absences, notify Class Teacher + Dean + Parent |
| `timetable.createEvents` | Timetable published | Create CalendarEvent rows for each slot for each student/teacher |
| `assignment.createEvent` | Assignment published | Create CalendarEvent (due date) for all class members |
| `quiz.createEvent` | Quiz published | Create CalendarEvent (quiz time) for all class members |

---

## Folder Structure (Sprint 003 additions)

```
app/(school)/[schoolSlug]/
├── academic/
│   ├── timetable/
│   │   ├── page.tsx            ← Dean: timetable designer
│   │   └── [classId]/page.tsx  ← Teacher/Student: view timetable
│   ├── attendance/
│   │   ├── page.tsx            ← Class Teacher: mark register
│   │   └── reports/page.tsx    ← Dean: attendance reports
│   ├── assignments/
│   │   ├── page.tsx            ← Teacher: list + create
│   │   ├── new/page.tsx
│   │   ├── [id]/page.tsx       ← Teacher: detail + submissions
│   │   └── [id]/submit/page.tsx ← Student: submit
│   ├── quizzes/
│   │   ├── page.tsx            ← Teacher: list + create
│   │   ├── new/page.tsx
│   │   ├── [id]/page.tsx       ← Teacher: detail + results
│   │   ├── [id]/attempt/page.tsx ← Student: quiz taking UI
│   │   └── [id]/results/page.tsx ← Student: results view
│   ├── exams/
│   │   ├── page.tsx            ← Teacher: list; Dean: approval queue
│   │   ├── [id]/page.tsx       ← Enter results, view detail
│   │   └── results/page.tsx    ← Student/Parent: view results
│   ├── lesson-plans/
│   │   └── page.tsx
│   └── syllabus/
│       └── page.tsx
├── analytics/
│   ├── page.tsx                ← Role-aware analytics home
│   ├── student/page.tsx
│   ├── teacher/page.tsx
│   └── school/page.tsx
└── calendar/
    └── page.tsx                ← Unified calendar (all roles)

server/routers/
├── academic.ts                 ← timetable, attendance, assignment, quiz, exam, lessonPlan, syllabus
└── analytics.ts                ← all analytics procedures

components/
├── academic/
│   ├── timetable/TimetableGrid.tsx
│   ├── attendance/AttendanceRegister.tsx
│   ├── assignments/AssignmentCard.tsx
│   ├── quizzes/QuizAttemptUI.tsx    ← 'use client', timer, answer state
│   ├── quizzes/QuizBuilder.tsx      ← 'use client', drag-drop questions
│   └── exams/ResultsEntry.tsx
├── analytics/
│   ├── StudentScoreTrend.tsx        ← 'use client', Recharts LineChart
│   ├── SubjectRadar.tsx             ← 'use client', Recharts RadarChart
│   ├── AttendanceBar.tsx            ← 'use client', Recharts BarChart
│   ├── AssignmentDonut.tsx          ← 'use client', Recharts PieChart
│   └── ClassPerformanceTable.tsx    ← 'use client', sortable table
└── calendar/
    └── UnifiedCalendar.tsx          ← 'use client', react-big-calendar
```

---

## Critical Implementation Notes

### Quiz Timer (server-enforced)
The quiz has a `scheduledAt` and `durationMinutes`. The `QuizAttempt.startedAt` is set server-side. On submit:
```typescript
const timeAllowed = quiz.durationMinutes * 60 * 1000
const elapsed = Date.now() - attempt.startedAt.getTime()
const isAutoSubmit = elapsed > timeAllowed + (5 * 60 * 1000) // 5 min grace
```
If a student submits after `startedAt + durationMinutes + 5min`, the server still processes — but marks `isAutoSubmitted = true` and logs the overtime. The client timer is display-only. Never trust the client to enforce the deadline.

### Analytics Pre-computation (AnalyticsSummary)
Every time a score is saved, Inngest fires `analytics.recompute` with the affected `studentId`, `classId`, `subjectId`. The job:
1. Queries all ExamResults + QuizAttempts for that student/class/subject in the term
2. Computes: avg_score, highest_score, lowest_score, pass_rate, submission_rate, attendance_pct
3. Upserts into `AnalyticsSummary`

Chart queries ONLY hit `AnalyticsSummary` — never raw result tables. This keeps analytics fast even with 1,000+ students.

### Calendar Performance
`CalendarEvent` rows can grow large. Index on `(schoolId, startAt)`. The `getMyEvents` query filters:
- `visibilityScope = PERSONAL AND createdByUserId = currentUser`
- OR `visibilityScope = CLASS AND scopeEntityId IN userClassIds`
- OR `visibilityScope = SCHOOL AND schoolId = currentSchoolId`
- OR exists in `CalendarEventAttendee` for currentUser

Use a DB view `user_calendar_events` to pre-join this logic for performance.

### Exam Paper Presigned URL
```typescript
// Only generate URL if: role authorised AND (paper not yet uploaded OR releasedAt <= now())
async function getExamPaperUrl(examId: string, role: UserRole): Promise<string | null> {
  const exam = await prisma.exam.findUnique({ where: { id: examId } })
  const canViewUnreleased = ['DEAN_ACADEMICS','HEADTEACHER'].includes(role)
  const isReleased = exam.paperReleasedAt && new Date() >= exam.paperReleasedAt
  
  if (!canViewUnreleased && !isReleased) return null
  if (!exam.paperS3Key) return null
  
  return generatePresignedGetUrl(exam.paperS3Key, 900) // 15 min expiry
}
```
