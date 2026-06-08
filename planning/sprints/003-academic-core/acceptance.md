# Acceptance Criteria — Sprint 003: Academic Core

**Linked to:** requirements.md  
**Last updated:** 2026-05-29

All criteria must pass before Sprint 004 begins.

---

## AC-001: Timetable

**Given** Dean is on the Timetable Designer page for Class 7A, Term 1  
**When** they add a slot: Maths, Teacher Kamau, Room 3, Monday Period 1  
**Then** the slot appears in the timetable grid and is saved to `TimetableSlot`

**Given** Dean adds a second slot for Teacher Kamau on Monday Period 1 (different class)  
**When** they attempt to save  
**Then** a conflict error is shown: "Teacher Kamau is already scheduled at Monday Period 1" — save blocked

**Given** Dean publishes the timetable  
**When** publish is confirmed  
**Then** `CalendarEvent` rows are created for every slot for every student in Class 7A and for Teacher Kamau

**Test type:** Integration

---

## AC-002: Attendance

**Given** Class Teacher Odhiambo is on the attendance register for Class 5B, today's date  
**When** they click "Mark All Present" then change 2 students to Absent  
**Then** 48 PRESENT + 2 ABSENT `AttendanceRecord` rows exist for today's date and classId

**Given** a student has been absent for 3 consecutive school days  
**When** the nightly Inngest job runs  
**Then** Class Teacher, Dean, and Parent receive an in-app notification with student name and absence count

**Given** a Class Teacher tries to mark attendance for a date 8 days ago  
**When** the form submits  
**Then** tRPC returns: `BAD_REQUEST: "Cannot mark attendance more than 7 days in the past"`

**Test type:** Integration + Inngest job test

---

## AC-003: Assignments

**Given** Teacher creates an assignment: "Essay on Kenya", due tomorrow, max 100 marks  
**When** it is published  
**Then** all students in the class see it on their dashboard with status PENDING, and a CalendarEvent is created for tomorrow's due date

**Given** Student submits a text response  
**When** submission is saved  
**Then** `AssignmentSubmission.status = SUBMITTED`, `submittedAt` is set, `isLate = false`

**Given** Teacher marks the submission with score 78 and feedback "Good work"  
**When** marking is saved  
**Then** student receives in-app notification; `submission.score = 78`, `submission.status = MARKED`

**Given** it is past the due date and `allowLateSubmission = true, latePenaltyPercent = 10`  
**When** a student submits  
**Then** `isLate = true`, and when teacher marks, the UI shows: "Raw: 70 → After penalty: 63"

**Test type:** E2E (Playwright)

---

## AC-004: Quizzes

**Given** Teacher creates a 3-question quiz: 2 MCQ + 1 True/False, 10 minutes, scheduled now  
**When** quiz is published  
**Then** students in the class see "Start Quiz" button; CalendarEvent created for quiz time

**Given** Student starts quiz and selects answers within the time limit  
**When** they click Submit  
**Then** `QuizAttempt.status = SUBMITTED`; Inngest `quiz.autoGrade` job is queued

**Given** Inngest auto-grade job runs  
**When** grading completes (all MCQ + True/False)  
**Then** `QuizAttempt.totalScore`, `percentageScore`, `isPassed` are computed and saved; `QuizAnswer.isCorrect` set on each auto-gradeable answer

**Given** a student's time expires (durationMinutes elapsed + 5min grace)  
**When** the server receives any subsequent answer save attempt  
**Then** tRPC returns `BAD_REQUEST: "Quiz time has expired"` and attempt is auto-submitted

**Given** Teacher releases results  
**When** student views results page  
**Then** student sees: total score, percentage, pass/fail, per-question breakdown with correct answer and explanation

**Test type:** Integration + Inngest job test

---

## AC-005: Exams & Results

**Given** Teacher creates an exam, enters results for all 30 students  
**When** results are submitted for Dean approval  
**Then** `exam.status = SUBMITTED`; Dean sees it in approval queue

**Given** Dean approves the exam  
**When** approval is confirmed  
**Then** `exam.status = APPROVED`, `approvedAt` set; students/parents do NOT yet see results

**Given** Dean releases the exam  
**When** release is confirmed  
**Then** `exam.status = RELEASED`, `releasedAt` set; students and parents receive notification; results are visible to students and parents

**Given** student requests exam paper presigned URL before `paperReleasedAt`  
**When** the tRPC call is made  
**Then** returns `null` — no URL generated

**Test type:** Integration

---

## AC-006: Analytics

**Given** 3 exam results and 2 quiz attempts exist for Student Achieng in Term 1  
**When** `analytics.recompute` Inngest job runs  
**Then** `AnalyticsSummary` has rows for: `avg_score`, `pass_rate` scoped to this student + term

**Given** Student Achieng visits her analytics page  
**When** the page loads  
**Then** LineChart shows score trend across the 5 assessments; RadarChart shows scores by subject; no raw DB queries — all from `AnalyticsSummary`

**Given** Parent visits their child's analytics page  
**When** the page loads  
**Then** they see the same charts as the student (read-only), PLUS class average comparison bar chart

**Given** Dean visits school analytics  
**When** the page loads  
**Then** all charts render in <800ms (verified by Playwright performance timing)

**Test type:** Integration (data correctness) + E2E (UI rendering + performance)

---

## AC-007: Unified Calendar

**Given** Teacher Kamau is logged in  
**When** they open the Calendar page  
**Then** they see: their timetable slots for the week, published assignment due dates for their classes, scheduled quiz times, approved exam dates — all colour-coded

**Given** Teacher Kamau creates a personal event "Staff meeting prep" tomorrow at 14:00  
**When** it is saved  
**Then** it appears ONLY on Teacher Kamau's calendar, not on other users' calendars

**Given** Dean publishes a school-wide event "Sports Day" next Saturday  
**When** any user opens their calendar  
**Then** they see "Sports Day" as an all-day event in orange

**Given** a user clicks the "Export .ics" button  
**When** the download completes  
**Then** the .ics file contains valid RFC 5545 format events for the next 30 days visible to that user

**Given** a user on the calendar switches between Month → Week → Day views  
**When** they switch back to Month  
**Then** their scroll position and selected date are preserved (`<Activity>` component)

**Test type:** E2E (Playwright)

---

## AC-008: Lesson Plans & Syllabus

**Given** Teacher creates a lesson plan for "Algebra - Quadratic Equations", saves as draft  
**When** draft is saved  
**Then** plan is visible to Teacher and Dean, NOT to students

**Given** Teacher publishes the lesson plan  
**When** published  
**Then** students in that class see a "Upcoming Lesson" note on their dashboard

**Given** Dean uploads a syllabus file for Mathematics, Grade 8, Term 1 and adds 12 topics  
**When** upload completes  
**Then** all Mathematics teachers for Grade 8 see the syllabus; students in Grade 8 see the topic list

**Given** Teacher marks Topic 3 as "Taught"  
**When** saved  
**Then** students see Topic 3 marked with a green tick; the syllabus shows "3/12 topics taught (25%)"

**Test type:** Integration

---

## Regression Baseline (Sprint 002 must still pass)

- All Clerk auth flows still working
- All 16 role dashboard shells still rendering
- RLS cross-tenant isolation still enforced
- School onboarding wizard still functional
- `// ARCH-QUESTION:` count = 0 before closing sprint

---

## Definition of Done

- All AC-001 through AC-008 pass
- `AnalyticsSummary` populated correctly from real data
- Calendar shows correct events for 3 tested roles (Teacher, Student, Parent)
- Quiz timer enforced server-side (integration test proves late submit rejected)
- All Inngest jobs tested with `inngest-cli dev` locally
- Prisma migrations for Sprint 003 entities deployed to staging Neon branch
- RLS policies added for all new tables (attendance, quiz, exam, calendar entities)
- `CONTEXT.md` Sprint History updated
- `STATE.md` updated
