# Requirements — Sprint 003: Academic Core

**Project:** EduManage  
**Sprint:** 003  
**Status:** Approved — ready for Builder  
**Last updated:** 2026-05-29  
**Depends on:** Sprint 002 complete (auth, tenancy, all 16 roles live)

---

## Sprint Goal

Deliver the primary daily-use academic features so that teachers can manage classes, record attendance, set and mark assignments, run quizzes, enter exam results — and students, parents, and teachers can view performance analytics — all within a unified calendar. After this sprint, the platform has enough value to demonstrate to pilot schools.

---

## Feature Modules in This Sprint

1. **Timetable & Scheduling** (Dean of Academics)
2. **Attendance Tracking** (Class Teacher)
3. **Assignments** — set, submit, mark, feedback (Teacher ↔ Student)
4. **Quizzes** — create, attempt, auto-grade, results (Teacher ↔ Student)
5. **Exams & Results** — entry, Dean approval, release (Teacher → Dean → Student/Parent)
6. **Performance Analytics** — student, teacher, parent views (Recharts)
7. **Unified Calendar** — personal + school + class + club events (all 16 roles)
8. **Lesson Plans** (Teacher)
9. **Syllabus Management** (Dean → Teacher → Student)

---

## FR-001: Timetable & Scheduling

**Priority:** P0  
**Roles:** Dean creates; all users view

### What it does
- Dean designs the school timetable: Subject → Class → Teacher → Time Slot → Room
- CBC and IGCSE period structures supported (CBC: 8 periods × 40min; IGCSE: 6 periods × 50min, configurable)
- On save: conflict detection runs — same teacher double-booked, same room double-booked, class has >2 consecutive same-subject periods
- Published timetable auto-populates into all affected users' personal calendars
- Dean can assign substitute teacher when leave is approved (Sprint 005) — substitution updates timetable + calendar

### Data
```
Timetable { id, schoolId, termId, classId, name, isPublished }
TimetableSlot { id, timetableId, schoolId, subjectId, teacherId, roomId, dayOfWeek, periodNumber, startTime, endTime }
Room { id, schoolId, name, capacity, building }
```

### Business rules
- Only Dean can create/edit timetable
- Timetable must be published before it appears on calendars
- Cannot have >6 periods per day per class (configurable by school)
- Teacher cannot be in two slots at the same time (hard constraint — blocked on save)
- Room cannot be double-booked (soft warning — allowed with override)

---

## FR-002: Attendance Tracking

**Priority:** P0  
**Roles:** Class Teacher marks; Dean/Headteacher view reports; Parent/Student view own

### What it does
- Class Teacher opens daily register for their class
- Mark each student: Present / Absent / Late / Excused
- Bulk action: "Mark all Present" then edit exceptions (most common workflow)
- Attendance can be edited up to 24 hours after submission
- Attendance summary per student per term (%)
- Alert: Inngest job runs nightly — students with ≥3 consecutive absences → notification to Class Teacher + Dean + Parent

### Data
```
AttendanceRecord { id, schoolId, classId, studentId, date, status (PRESENT|ABSENT|LATE|EXCUSED), notes?, markedByTeacherId, createdAt }
```

### Business rules
- Only one attendance record per student per day per class
- Cannot mark attendance for a date in the future
- Cannot mark attendance for a date >7 days ago (unless IT Support override)
- Attendance % = (Present + Late) / Total school days × 100

---

## FR-003: Assignments

**Priority:** P0  
**Roles:** Teacher creates/marks; Student submits; Parent views child's status

### What it does

**Teacher workflow:**
1. Creates assignment: title, description, subject, class, due date, max marks, file attachment (optional, S3)
2. Assignment appears in students' dashboards and calendar
3. After due date: Teacher views submissions list, marks each (score + written feedback)
4. Marks saved → Student notified → Parent notified

**Student workflow:**
1. Sees assignment on dashboard + calendar
2. Submits: text response OR file upload (S3, max 10MB)
3. Late submissions flagged but allowed (teacher can configure late penalty)
4. After marking: sees score + teacher feedback

### Data
```
Assignment {
  id, schoolId, subjectId, classId, teacherId, title, description,
  dueDate, maxMarks, allowLateSubmission, latePenaltyPercent,
  attachmentUrl?, status (DRAFT|PUBLISHED|CLOSED), createdAt
}

AssignmentSubmission {
  id, schoolId, assignmentId, studentId,
  submittedAt, isLate, textResponse?, attachmentUrl?,
  score?, feedback?, markedAt?, markedByTeacherId?
  status (PENDING|SUBMITTED|MARKED|RETURNED)
}
```

### Business rules
- Submission after `dueDate` = `isLate = true`; score reduced by `latePenaltyPercent` automatically
- Teacher can override late penalty on individual submission
- Student cannot edit submission after teacher has started marking
- Parent sees: assignment title, due date, submission status, score (after marking) — no file access
- Assignment in DRAFT not visible to students

---

## FR-004: Quizzes

**Priority:** P0  
**Roles:** Teacher creates; Student attempts; Teacher/Dean view results

### What it does

**Teacher workflow:**
1. Creates quiz: title, subject, class, scheduled date/time, duration (minutes), attempts allowed
2. Adds questions (see types below)
3. Publishes — appears in students' calendar at scheduled time
4. After close: results auto-calculated for MCQ/True-False; Short Answer flagged for manual grading
5. Teacher reviews short answers, enters manual scores

**Student workflow:**
1. Quiz appears in calendar. At scheduled time, "Start Quiz" button activates.
2. Timer counts down. Auto-submits when time expires.
3. Can navigate between questions, flag for review, change answers until submit
4. After teacher releases results: sees score, correct answers, feedback per question

### Question Types (DEC-015)

| Type | Auto-grade | How |
|---|---|---|
| MCQ (Multiple Choice) | ✅ Yes | Exact match to `correctOptionIndex` |
| True/False | ✅ Yes | Exact match to `correctAnswer` boolean |
| Short Answer | ❌ No | Teacher reviews, enters score manually |
| Essay | ❌ No (v2 AI) | Deferred to Sprint 007 |

### Data
```
Quiz {
  id, schoolId, subjectId, classId, teacherId,
  title, description, scheduledAt, durationMinutes,
  attemptsAllowed (default 1), isPublished, resultsReleased,
  totalMarks (computed), createdAt
}

QuizQuestion {
  id, quizId, schoolId, questionText, questionType,
  options Json?,           -- ["Option A","Option B","Option C","Option D"]
  correctOptionIndex Int?, -- MCQ: 0-3
  correctAnswer Boolean?,  -- True/False
  marks Int,
  orderIndex Int,
  explanation String?      -- shown after results released
}

QuizAttempt {
  id, quizId, studentId, schoolId,
  startedAt, submittedAt, isAutoSubmitted,
  totalScore?, percentageScore?, isPassed?,
  status (IN_PROGRESS|SUBMITTED|GRADED)
}

QuizAnswer {
  id, attemptId, questionId, schoolId,
  selectedOptionIndex Int?,  -- MCQ answer
  selectedBoolean Boolean?,  -- True/False answer
  textAnswer String?,        -- Short answer text
  isCorrect Boolean?,        -- null for short answer until marked
  marksAwarded Int?,
  teacherFeedback String?
}
```

### Business rules
- Student can only start quiz during the active window (`scheduledAt` to `scheduledAt + durationMinutes + grace 5min`)
- Timer is server-enforced — client timer is display only
- Auto-grade runs via Inngest job on submission (not synchronously)
- Results not visible to students until teacher sets `resultsReleased = true`
- Short answer questions require teacher manual grading before results can be released
- `isPassed` = `percentageScore >= quiz.passMark` (default 50%)

---

## FR-005: Exams & Results

**Priority:** P0  
**Roles:** Teacher enters results; Dean approves; Student/Parent view after release

### What it does
- Teacher creates exam record: name, subject, class, date, max marks, exam type (CAT | MID_TERM | END_TERM | MOCK | PRACTICAL)
- Teacher uploads exam paper (.docx) with password entry on upload (DEC-011)
- Teacher enters student results per exam per subject
- Dean approves results → bulk release → students and parents notified
- CBC grading: Exceeding / Meeting / Approaching / Below (mapped from score ranges set by school)
- IGCSE grading: A* A B C D E F G U (mapped from score ranges)
- Class performance metrics: average, highest, lowest, pass rate (auto-computed on approval)

### Data
```
Exam {
  id, schoolId, classId, subjectId, teacherId,
  title, examType, examDate, maxMarks,
  paperS3Key?, paperReleasedAt?,    -- DEC-011 encrypted paper
  status (DRAFT|SUBMITTED|APPROVED|RELEASED), approvedByDeanId?,
  approvedAt?, releasedAt?, createdAt
}

ExamResult {
  id, examId, studentId, schoolId,
  rawScore, grade (CBC or IGCSE enum), isAbsent,
  teacherComment?, createdAt
}
```

### Business rules
- Exam paper: S3 key stored encrypted; presigned URL only generated after `paperReleasedAt <= now()` for authorised roles
- Dean must approve before release; Headteacher can also approve
- After approval, results are immutable (teacher must request Dean unlock to edit)
- Grade computed automatically from `rawScore` and school's grade boundary config

---

## FR-006: Performance Analytics

**Priority:** P0  
**Roles:** Student (own), Teacher (class), Parent (own child), Dean/Headteacher (school-wide)

This is a core differentiator — every role gets a tailored analytics view using Recharts.

### Student Analytics Dashboard (`/[schoolSlug]/analytics/student`)

| Chart | Type | Description |
|---|---|---|
| Score Trend | Line | Exam/quiz scores over time per subject |
| Subject Radar | Radar | Performance across all subjects this term |
| Attendance Bar | Bar | Monthly attendance % |
| Assignment Completion | Donut | Submitted / Late / Missing this term |
| Class Rank | Single stat | Student rank within class (optional, school configurable) |

### Teacher Analytics Dashboard (`/[schoolSlug]/analytics/teacher`)

| Chart | Type | Description |
|---|---|---|
| Class Average Trend | Line | Class average per exam/quiz over time |
| Subject Performance | Grouped Bar | Current term vs last term per assessment |
| Student Performance Table | Table | All students ranked; flag top/bottom 10% |
| Submission Rate | Donut | On-time / Late / Missing for each assignment |
| Attendance Overview | Bar | Class attendance % per month |

### Parent Analytics Dashboard (`/[schoolSlug]/analytics/parent`)

| Chart | Type | Description |
|---|---|---|
| Child's Score Trend | Line | Same as student but read-only |
| Subject Comparison | Bar | Child vs class average per subject |
| Attendance Trend | Bar | Monthly attendance |
| Assignment Status | List | Recent assignments + status + score |

### Dean / Headteacher Analytics (`/[schoolSlug]/analytics/school`)

| Chart | Type | Description |
|---|---|---|
| School-wide Performance | Grouped Bar | Average per class per exam type |
| Top/Bottom Classes | Bar | Ranked classes by term average |
| Teacher Performance | Table | Class averages per teacher |
| Attendance Heatmap | Calendar heatmap | School-wide attendance by day |
| Pass Rate Trend | Line | % passing per term |

### Analytics Data Layer

- Source of truth: `ExamResult`, `QuizAttempt`, `AttendanceRecord`, `AssignmentSubmission` tables
- Pre-computed summaries in `AnalyticsSummary` table updated by Inngest job after each exam/quiz graded
- Chart queries go to summaries, not raw tables (performance)
- All analytics scoped by `schoolId` (RLS) + role (what you can see)

```
AnalyticsSummary {
  id, schoolId, entityType (STUDENT|CLASS|SUBJECT|SCHOOL),
  entityId, termId, metric, value (Decimal), computedAt
}
```

---

## FR-007: Unified Calendar

**Priority:** P0  
**Roles:** All 16 roles — each sees a personalised calendar

### What it does

Every user has one calendar view that aggregates:

| Event Source | Who sees it | Colour |
|---|---|---|
| Timetable slots | Teacher (own classes), Student (own classes) | Teal |
| Assignment due dates | Teacher (own assignments), Student (own) | Violet |
| Quiz scheduled times | Teacher (own quizzes), Student (own) | Blue |
| Exam dates | All school staff + students | Red |
| School events (holidays, meetings) | All | Orange |
| Class events | Class members | Amber |
| Club events | Club members | Green |
| Trip events | Involved users | Sky |
| Leave requests | Dean, affected teacher | Gray |
| Personal events | Creator only | Slate |

### Views
- Month view (default)
- Week view (shows timetable slots clearly)
- Day view (detailed hourly)
- Agenda view (list of upcoming events, 7 days)

### Interactions
- Click event → event detail sheet (no route change)
- Create personal event: "+" button → inline form
- Export to `.ics` (Google Calendar / Apple Calendar)
- Filter by event type (toggle chips)
- `<Activity>` component preserves view/date when user switches tabs

### Data
```
CalendarEvent {
  id, schoolId, title, description?,
  startAt, endAt, isAllDay,
  eventType (TIMETABLE|ASSIGNMENT|QUIZ|EXAM|SCHOOL|CLASS|CLUB|TRIP|LEAVE|PERSONAL),
  sourceId?,       -- FK to originating record (timetable slot, assignment, etc.)
  sourceType?,     -- "TimetableSlot" | "Assignment" | etc.
  createdByUserId, visibilityScope (PERSONAL|CLASS|SCHOOL|CLUB),
  scopeEntityId?,  -- classId | clubId for scoped events
  colour?,
  createdAt
}

CalendarEventAttendee {
  eventId, userId, schoolId, status (INVITED|ACCEPTED|DECLINED)
}
```

### Business rules
- Timetable slots auto-create CalendarEvents when timetable is published
- Assignment creation auto-creates a CalendarEvent (due date) for all class members
- Quiz scheduling auto-creates a CalendarEvent for all class members
- Exam date creation auto-creates a CalendarEvent for class + subject teacher
- School-wide events visible to all users in the school
- Personal events visible to creator only
- `.ics` export includes only events visible to the requesting user

---

## FR-008: Lesson Plans

**Priority:** P1  
**Roles:** Teacher creates; Dean views all

- Teacher creates lesson plan: date, class, subject, topic, objectives, activities, resources needed
- Save as draft or publish to class (published = visible to students as upcoming lesson note)
- Download as .docx (standard template)
- Dean can view all teachers' lesson plans per subject

### Data
```
LessonPlan {
  id, schoolId, teacherId, classId, subjectId,
  date, topic, objectives Json, activities Json,
  resourcesNeeded String?, status (DRAFT|PUBLISHED),
  docxS3Key?, createdAt
}
```

---

## FR-009: Syllabus Management

**Priority:** P1  
**Roles:** Dean uploads; Teachers + Students view

- Dean uploads syllabus per subject per class (.pdf or .docx, S3)
- Syllabus broken into topics (Dean enters as a list)
- Teacher marks topic as "taught" on their class syllabus
- Student views syllabus with teacher's progress markers (% taught)

### Data
```
Syllabus { id, schoolId, subjectId, classId, termId, fileS3Key, uploadedByDeanId, createdAt }
SyllabusTopic { id, syllabusId, schoolId, title, description?, orderIndex }
SyllabusProgress { syllabusTopicId, classId, teacherId, schoolId, taughtAt? }
```

---

## Non-Functional Requirements

- Analytics charts render <800ms on Standard tier (pre-computed summaries)
- Calendar with 500 events loads in <1 second (virtualised event rendering)
- Quiz timer is accurate to within ±2 seconds (server-validated on submit)
- Attendance bulk mark + save completes in <500ms for a class of 50
- All pages fully responsive — mobile-first (majority of Kenyan users on phone)

---

## Dashboard Tabs Delivered This Sprint

**Dean:** Timetable, Exams (approval queue), Analytics (school-wide), Calendar  
**Class Teacher:** My Class, Attendance, Assignments, Quizzes, Exams, Lesson Plans, Analytics, Calendar  
**Subject Teacher:** My Subjects, Assignments, Quizzes, Exams, Lesson Plans, Calendar  
**Student:** Dashboard, Assignments, Quizzes, My Results, Analytics, Calendar  
**Parent:** Child's Results, Child's Assignments, Child's Quizzes, Analytics, Calendar  
**Headteacher:** Analytics (school-wide), Calendar, Exam Approvals

---

## Out of Scope (this sprint)

- Fee payments (Sprint 004)
- Messaging system (Sprint 005)
- Library resources (Sprint 006)
- AI-generated reports (Sprint 007)
- Boarding, transport, nursing, kitchen, HR (Sprint 008)
- Club calendar events (Sprint 005)
- Leave management (Sprint 005)
