# API_REFERENCE.md — EduManage tRPC Router Reference

**Last updated:** 2026-05-29  
**API Layer:** tRPC v11 over Next.js App Router  
**Endpoint:** `POST /api/trpc/[procedure]`  
**Auth:** All procedures require valid Clerk JWT (except public routes)

---

## Conventions

```typescript
// Error codes
UNAUTHORIZED    // No valid JWT
FORBIDDEN       // Valid JWT, wrong role or wrong school
BAD_REQUEST     // Invalid input (Zod validation failure)
NOT_FOUND       // Entity doesn't exist or RLS filtered it out
INTERNAL_SERVER_ERROR // Unexpected server error

// Role notation (shorthand used in this doc)
SA = SUPER_ADMIN | HT = HEADTEACHER | DEAN = DEAN_ACADEMICS
CT = CLASS_TEACHER | ST = SUBJECT_TEACHER | ACCT = ACCOUNTANT
PROC = PROCUREMENT_OFFICER | IT = IT_SUPPORT | HR = HR_MANAGER
TM = TRANSPORT_MANAGER | LIB = LIBRARIAN | NURSE = SCHOOL_NURSE
CAT = CATERESS | MAT = MATRON | PAR = PARENT | STU = STUDENT
```

---

## `school.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `school.create` | SA | Create new school + Clerk org + invite Headteacher |
| `school.list` | SA | All schools with subscription status |
| `school.getBySlug` | All (own school) | School profile data |
| `school.update` | HT | Update name, logo, email domain |
| `school.onboarding.saveStep` | HT | Save wizard step (1–5) |
| `school.onboarding.getState` | HT | Resume wizard |
| `school.inviteUser` | HT, IT | Invite staff member + assign role |
| `school.deactivateUser` | HT, IT | Remove from Clerk org + set isActive=false |
| `school.changeRole` | HT | Update user role in Clerk metadata + DB |
| `school.listUsers` | HT, IT, HR | All users in school |

---

## `academic.*` Router

### Timetable
| Procedure | Roles | Description |
|---|---|---|
| `academic.timetable.create` | DEAN | Create timetable for a class/term |
| `academic.timetable.addSlot` | DEAN | Add slot (subject+teacher+room+time) |
| `academic.timetable.removeSlot` | DEAN | Remove slot |
| `academic.timetable.detectConflicts` | DEAN | Sync conflict check — returns ConflictResult[] |
| `academic.timetable.publish` | DEAN | Publish + trigger calendar event creation |
| `academic.timetable.getByClass` | All | View timetable for a class |

### Attendance
| Procedure | Roles | Description |
|---|---|---|
| `academic.attendance.bulkMark` | CT | Mark/update daily attendance for class |
| `academic.attendance.getByClass` | CT, DEAN, HT | Class attendance for a date |
| `academic.attendance.getByStudent` | STU(own), PAR(child), CT, DEAN | Student attendance history |
| `academic.attendance.getSummary` | CT, DEAN, HT | % summary per student per term |

### Assignments
| Procedure | Roles | Description |
|---|---|---|
| `academic.assignment.create` | CT, ST | Create assignment |
| `academic.assignment.publish` | CT, ST (own) | Publish → notify students |
| `academic.assignment.close` | CT, ST (own) | Close submissions |
| `academic.assignment.submit` | STU (own class) | Submit assignment |
| `academic.assignment.mark` | CT, ST (own) | Score + feedback |
| `academic.assignment.listForClass` | CT, ST, DEAN | All assignments for a class |
| `academic.assignment.listForStudent` | STU(own), PAR(child) | Student's assignments |
| `academic.assignment.getSubmissions` | CT, ST (own) | All submissions for an assignment |

### Quizzes
| Procedure | Roles | Description |
|---|---|---|
| `academic.quiz.create` | CT, ST | Create quiz |
| `academic.quiz.addQuestion` | CT, ST (own quiz) | Add MCQ/T-F/Short Answer question |
| `academic.quiz.publish` | CT, ST (own quiz) | Publish → notify students |
| `academic.quiz.startAttempt` | STU (own class, within window) | Start timed attempt |
| `academic.quiz.saveAnswer` | STU (own attempt, within time) | Save answer (auto-saved) |
| `academic.quiz.submitAttempt` | STU (own attempt) | Final submission → triggers autograde |
| `academic.quiz.gradeShortAnswer` | CT, ST (own quiz) | Manually grade short-answer |
| `academic.quiz.releaseResults` | CT, ST (own quiz) | Release results to students |
| `academic.quiz.getResults` | CT, ST (own), STU (own, after release), PAR (child, after release), DEAN | Results |

### Exams
| Procedure | Roles | Description |
|---|---|---|
| `academic.exam.create` | CT, ST | Create exam record |
| `academic.exam.getPaperUploadUrl` | CT, ST (own) | Presigned S3 URL for paper upload |
| `academic.exam.enterResults` | CT, ST (own) | Bulk result entry |
| `academic.exam.approve` | DEAN, HT | Approve results for release |
| `academic.exam.release` | DEAN, HT | Release to students/parents |
| `academic.exam.getResults` | CT, ST, DEAN, HT (all); STU/PAR (own, after release) | Results |
| `academic.exam.getPaperUrl` | DEAN, HT, CT, ST (own, before release); all (after release) | Presigned paper URL |

### Lesson Plans & Syllabus
| Procedure | Roles | Description |
|---|---|---|
| `academic.lessonPlan.create` | CT, ST | Create lesson plan |
| `academic.lessonPlan.publish` | CT, ST (own) | Publish to class |
| `academic.lessonPlan.list` | CT, ST (own), DEAN | List lesson plans |
| `academic.syllabus.upload` | DEAN | Upload syllabus file + topics |
| `academic.syllabus.addTopics` | DEAN | Add topic list |
| `academic.syllabus.markTaught` | CT, ST | Mark topic as taught |
| `academic.syllabus.getByClass` | CT, ST, STU (own class), DEAN | View syllabus with progress |

---

## `analytics.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `analytics.getStudentAnalytics` | STU(own), PAR(child), CT, ST, DEAN, HT | Charts data for a student |
| `analytics.getClassAnalytics` | CT(own), ST(own subject), DEAN, HT | Class performance data |
| `analytics.getSchoolAnalytics` | DEAN, HT | School-wide performance |
| `analytics.getTeacherAnalytics` | CT(own), ST(own), DEAN, HT | Teacher's class averages |

---

## `calendar.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `calendar.getMyEvents` | All | Personalised events for date range |
| `calendar.createPersonalEvent` | All | Private personal event |
| `calendar.createSchoolEvent` | HT, DEAN | Visible to all school users |
| `calendar.createClassEvent` | CT, DEAN | Visible to class members |
| `calendar.exportIcs` | All | .ics file for date range (own events only) |

---

## `financial.*` Router

### Fees
| Procedure | Roles | Description |
|---|---|---|
| `financial.fee.configureStructure` | ACCT | Create/update fee structure |
| `financial.fee.approveStructure` | HT | Approve fee structure |
| `financial.fee.getStudentStatement` | ACCT, PAR(child), STU(own) | Fee statement + balance |
| `financial.fee.initiatePayment` | PAR(own child), ACCT | STK Push initiation |
| `financial.fee.recordManualPayment` | ACCT | Cash/bank/cheque entry |
| `financial.fee.downloadStatement` | ACCT, PAR(own child) | PDF/CSV statement |

### Payroll
| Procedure | Roles | Description |
|---|---|---|
| `financial.payroll.createRun` | ACCT | Create salary run for month |
| `financial.payroll.enterSalaries` | ACCT | Enter/confirm staff salaries |
| `financial.payroll.calculatePAYE` | ACCT | Preview PAYE for a gross amount |
| `financial.payroll.approveRun` | HT | Approve salary run |
| `financial.payroll.generatePayslips` | ACCT | Trigger payslip generation |

### Tax & Bank
| Procedure | Roles | Description |
|---|---|---|
| `financial.tax.generatePAYE` | ACCT | Generate P10 CSV for iTax |
| `financial.tax.generateVAT` | ACCT | Generate VAT3 CSV |
| `financial.tax.downloadReport` | ACCT | Download tax report |
| `financial.bank.uploadStatement` | ACCT | Parse + import bank CSV |
| `financial.bank.manualMatch` | ACCT | Match transaction to payment |

### Reports & Subscriptions
| Procedure | Roles | Description |
|---|---|---|
| `financial.reports.generate` | ACCT, HT | Generate financial report |
| `financial.reports.download` | ACCT, HT | Download generated report |
| `financial.subscription.getStatus` | HT, SA | Current subscription details |
| `financial.subscription.initiateStripeCheckout` | HT, SA | Create Stripe Checkout session |
| `financial.subscription.initiateRenewalMpesa` | HT | STK Push for subscription renewal |

---

## `messaging.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `messaging.getThreads` | All | User's conversation list |
| `messaging.getMessages` | Thread member | Paginated message history |
| `messaging.sendMessage` | Thread member | Send message + attachment |
| `messaging.createDirect` | All | Get or create 1-to-1 thread |
| `messaging.createGroup` | HT, DEAN, CT | Create group thread |
| `messaging.markRead` | Thread member | Update lastReadAt |
| `messaging.archiveThread` | Thread member | Soft-delete from user's list |

---

## `notifications.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `notifications.getMyNotifications` | All | Paginated notifications (newest first) |
| `notifications.markRead` | Own notifications | Mark one or all as read |
| `notifications.getUnreadCount` | All | Badge count |
| `notifications.getPreferences` | All | Email/in-app preferences per type |
| `notifications.updatePreferences` | All | Toggle email/in-app per type |

---

## `clubs.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `clubs.create` | HT, DEAN | Create new club |
| `clubs.list` | All | All active clubs |
| `clubs.join` | STU | Join (immediate or pending) |
| `clubs.approveJoin` | Patron | Approve pending member |
| `clubs.createEvent` | Patron | Create club event |
| `clubs.logProgress` | Patron | Record meeting notes |
| `clubs.requestBudget` | Patron | Submit budget request |
| `clubs.approveBudget` | ACCT, HT | Approve club budget |

---

## `leave.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `leave.request` | CT, ST | Submit leave request |
| `leave.getQueue` | DEAN | Pending leave requests |
| `leave.review` | DEAN | Approve/reject + assign substitute |
| `leave.getMyHistory` | Own (teacher) | Own leave history |
| `leave.getSchoolOverview` | HT, DEAN | All leaves summary |

---

## `procurement.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `procurement.create` | All staff | Create procurement request |
| `procurement.submit` | Requestor | Submit DRAFT → PENDING_HOD |
| `procurement.approve` | Current step approver | Advance approval chain |
| `procurement.reject` | Any approver | Reject with note |
| `procurement.createOrder` | PROC | Place order with supplier |
| `procurement.confirmDelivery` | PROC | Mark delivered + update inventory |
| `procurement.reportDamage` | All staff | Log damaged items |
| `procurement.getInventory` | PROC, dept heads | View inventory |
| `procurement.updateInventory` | PROC | Update quantities |
| `procurement.listRequests` | PROC, HT, ACCT, own (requestor) | Filter by status/dept |

---

## `library.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `library.resource.getUploadUrl` | TEACHER, LIB | Presigned S3 POST (quota check first) |
| `library.resource.create` | TEACHER, LIB | Finalise resource after upload |
| `library.resource.review` | LIB | Approve or reject |
| `library.resource.search` | TEACHER, STU(own classes), LIB, DEAN, HT | Search APPROVED resources |
| `library.resource.getDownloadUrl` | TEACHER, STU(own classes), LIB, DEAN | Presigned GET URL |
| `library.resource.list` | LIB(all), TEACHER(own+APPROVED), DEAN | List resources |
| `library.student.getUploadUrl` | STU | Personal library upload (quota check) |
| `library.student.saveItem` | STU | Save personal item |
| `library.student.list` | STU(own), PAR(child — metadata only), LIB | Personal library |
| `library.student.createShareLink` | STU (own) | Generate shareable token |
| `library.student.accessByToken` | STU (same school) | Access shared item |
| `library.parent.getChildActivity` | PAR(own child) | Activity log — no file content |
| `library.parent.suggestResource` | PAR(own child) | Suggest URL to child |
| `library.student.getSuggestions` | STU | View parent suggestions |
| `library.storage.getUsage` | HT, LIB, IT | School storage usage |
| `library.storage.getBreakdown` | IT | Per-user breakdown |

---

## `ai.*` Router (Premium Only)

| Procedure | Roles | Description |
|---|---|---|
| `ai.reports.generateTermReports` | DEAN, HT | Trigger batch Gemini report generation |
| `ai.reports.list` | DEAN, HT, CT(own class) | List AIReport rows |
| `ai.reports.updateDraft` | DEAN, CT | Edit AI-generated text |
| `ai.reports.approve` | HT | Approve → notify parents |
| `ai.reports.publish` | HT | Publish → parent can view |
| `ai.insights.getActive` | HT, DEAN, ACCT, CT, ST | Unacknowledged insights |
| `ai.insights.acknowledge` | Same as above | Mark insight acknowledged |
| `ai.feedback.request` | CT, ST (own assignment) | Request feedback suggestions |
| `ai.feedback.getResult` | CT, ST (own) | Poll for generated suggestions |
| `ai.timetable.requestReview` | DEAN | Queue timetable AI review |
| `ai.timetable.getReviewResult` | DEAN | Poll for review results |
| `ai.lessonPlan.generateDraft` | CT, ST | Generate lesson plan draft |
| `ai.financial.generateNarrative` | ACCT, HT | Generate financial summary |
| `ai.usage.getStats` | HT, IT | Token usage dashboard |

---

## `boarding.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `boarding.createDorm` | MAT, HT | Create dorm |
| `boarding.updateDorm` | MAT, HT | Update dorm details |
| `boarding.listDorms` | MAT, HT | All dorms with occupancy |
| `boarding.getOccupancy` | MAT, HT | Bed-level occupancy view |
| `boarding.allocateBed` | MAT | Assign student to bed |
| `boarding.vacateBed` | MAT | Mark bed vacated |
| `boarding.transferStudent` | MAT | Move to different dorm/bed |
| `boarding.updateCleaningRoster` | MAT | Update cleaning assignments |
| `boarding.raiseProcurement` | MAT | Shortcut to procurement.create |

---

## `transport.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `transport.createVehicle` | TM | Register vehicle |
| `transport.updateVehicle` | TM | Update vehicle details |
| `transport.listVehicles` | TM, HT | All vehicles |
| `transport.planTrip` | All teachers, TM | Create trip request |
| `transport.updateManifest` | CT | Add students + consent |
| `transport.approveTrip` | Current approver | Advance approval chain |
| `transport.rejectTrip` | Any approver | Reject with note |
| `transport.completeTrip` | TM | Mark completed + actual cost |
| `transport.submitBudget` | TM | Term budget submission |
| `transport.approveBudget` | ACCT, HT | Approve transport budget |
| `transport.generateReport` | TM | PDF trip report |

---

## `nursing.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `nursing.logVisit` | NURSE | Log health visit |
| `nursing.getVisits` | **NURSE, HT ONLY** | View visit records |
| `nursing.flagFollowUp` | NURSE | Set follow-up date + notify |
| `nursing.getInventory` | NURSE | Medical inventory |
| `nursing.updateInventory` | NURSE | Update quantities |
| `nursing.raiseProcurement` | NURSE | Medical supply request |
| `nursing.logImmunisation` | NURSE | Log vaccination |
| `nursing.getImmunisations` | NURSE, HT | Student immunisation history |

---

## `kitchen.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `kitchen.createMealPlan` | CAT | Create weekly meal plan |
| `kitchen.addEntry` | CAT | Add meal per day/type |
| `kitchen.publishMealPlan` | CAT | Publish → all users see it |
| `kitchen.getInventory` | CAT | Kitchen inventory |
| `kitchen.updateInventory` | CAT | Update quantities |
| `kitchen.submitBudget` | CAT | Monthly budget submission |
| `kitchen.approveBudget` | ACCT, HT | Approve kitchen budget |
| `kitchen.generateMonthlyReport` | CAT | Monthly meals PDF |

---

## `hr.*` Router

| Procedure | Roles | Description |
|---|---|---|
| `hr.createRecord` | HR | Create staff record |
| `hr.updateRecord` | HR | Update staff details |
| `hr.getRecord` | HR, HT, own (basic info) | Staff record |
| `hr.uploadDocument` | HR | Upload staff document to S3 |
| `hr.getDocuments` | **HR, HT ONLY** | List staff documents |
| `hr.createAppraisal` | HR | Create annual appraisal |
| `hr.submitSelfAppraisal` | Own staff member | Self-appraisal text |
| `hr.submitLineManagerReview` | Line manager | Review text |
| `hr.approveAppraisal` | HT | Final approval |
| `hr.getOnboarding` | HR | Onboarding checklist |
| `hr.updateOnboarding` | HR | Mark checklist items |
| `hr.confirmPayrollHeadcount` | HR | Confirm active staff count → notify Accountant |
| `hr.getStaffDirectory` | All staff | Names, roles, departments |
| `hr.getLeaveOverview` | HR, HT | Consolidated leave summary |

---

## External Webhook Handlers (Not tRPC)

| Route | Method | Handler |
|---|---|---|
| `/api/webhooks/mpesa` | POST | Daraja C2B callback → Inngest |
| `/api/webhooks/stripe` | POST | Stripe events → Inngest |
| `/api/webhooks/clerk` | POST | Clerk org/user events → Inngest |
| `/api/inngest` | GET/POST/PUT | Inngest function registration |
