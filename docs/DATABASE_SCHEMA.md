# DATABASE_SCHEMA.md — EduManage Complete Schema Reference

**Last updated:** 2026-05-29  
**ORM:** Prisma 5  
**Database:** Neon (serverless Postgres)  
**Tenancy:** Row-Level Security — `school_id` on every table

---

## Schema Groups by Sprint

### Sprint 002 — Auth & Tenancy
`School` `User` `OnboardingState` `AuditLog`

### Sprint 003 — Academic Core
`Subject` `Room` `Timetable` `TimetableSlot`  
`AttendanceRecord`  
`Assignment` `AssignmentSubmission`  
`Quiz` `QuizQuestion` `QuizAttempt` `QuizAnswer`  
`Exam` `ExamResult`  
`AnalyticsSummary`  
`CalendarEvent` `CalendarEventAttendee`  
`LessonPlan` `Syllabus` `SyllabusTopic` `SyllabusProgress`

### Sprint 004 — Financial
`FeeStructure` `FeeItem` `StudentFeeBalance`  
`FeePayment` `MpesaStkRequest`  
`StaffSalaryRecord` `SalaryRun` `SalaryRunItem` `Payslip`  
`TaxRecord`  
`BankStatement` `BankTransaction`  
`SchoolSubscription` `FinancialReport`

### Sprint 005 — Communications & Procurement
`MessageThread` `ThreadMember` `Message`  
`Notification` `NotificationPreference`  
`Club` `ClubMembership` `ClubEvent` `ClubBudgetRequest`  
`LeaveRequest` `SubstituteAssignment`  
`ProcurementRequest` `ProcurementOrder` `InventoryItem` `DamageReport`

### Sprint 006 — Library
`LibraryResource`  
`StudentLibraryItem` `StudentLibraryActivity`  
`LibraryResourceSuggestion`  
`StorageUsage`

### Sprint 007 — AI
`AIReport` `AIInsight` `AIUsageLog`

### Sprint 008 — Departments
`Dorm` `DormAllocation` `CleaningRoster`  
`Vehicle` `Trip` `TripManifest` `TransportBudget`  
`HealthVisit` `MedicalInventoryItem` `ImmunisationRecord`  
`MealPlan` `MealPlanEntry` `KitchenInventoryItem` `KitchenBudget`  
`StaffRecord` `StaffDocument` `PerformanceAppraisal` `HROnboardingChecklist`

---

## Total Entity Count: ~75 models

---

## Universal RLS Pattern

Every model has:
```sql
ALTER TABLE [table_name] ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON [table_name]
  USING (school_id = current_setting('app.current_school_id', true)::uuid);
```

Exception — `HealthVisit` uses additional role check (see Sprint 008 blueprint).

---

## Key Enums (all sprints)

```prisma
enum CurriculumType       { CBC IGCSE BOTH }
enum SubscriptionTier     { BASIC STANDARD PREMIUM }
enum SubscriptionStatus   { TRIAL ACTIVE PAST_DUE SUSPENDED CANCELLED }
enum UserRole             { SUPER_ADMIN HEADTEACHER DEAN_ACADEMICS CLASS_TEACHER SUBJECT_TEACHER
                            ACCOUNTANT PROCUREMENT_OFFICER IT_SUPPORT HR_MANAGER TRANSPORT_MANAGER
                            LIBRARIAN SCHOOL_NURSE CATERESS MATRON PARENT STUDENT }
enum AttendanceStatus     { PRESENT ABSENT LATE EXCUSED }
enum AssignmentStatus     { DRAFT PUBLISHED CLOSED }
enum SubmissionStatus     { PENDING SUBMITTED MARKED RETURNED }
enum QuestionType         { MCQ TRUE_FALSE SHORT_ANSWER }
enum AttemptStatus        { IN_PROGRESS SUBMITTED GRADED }
enum ExamType             { CAT MID_TERM END_TERM MOCK PRACTICAL }
enum ExamStatus           { DRAFT SUBMITTED APPROVED RELEASED }
enum AnalyticsEntityType  { STUDENT CLASS SUBJECT SCHOOL }
enum CalendarEventType    { TIMETABLE ASSIGNMENT QUIZ EXAM SCHOOL CLASS CLUB TRIP LEAVE PERSONAL }
enum EventScope           { PERSONAL CLASS SCHOOL CLUB }
enum AttendeeStatus       { INVITED ACCEPTED DECLINED }
enum LessonPlanStatus     { DRAFT PUBLISHED }
enum FeeCategory          { TUITION ACTIVITY TRANSPORT MEALS BOARDING LIBRARY OTHER }
enum PaymentMethod        { MPESA BANK CASH CHEQUE }
enum PaymentStatus        { PENDING COMPLETED FAILED REVERSED }
enum SalaryStatus         { DRAFT APPROVED DISBURSED }
enum TaxType              { PAYE VAT WHT }
enum TaxStatus            { DRAFT REVIEWED FILED }
enum ReconcileStatus      { UNMATCHED MATCHED IGNORED }
enum NotificationType     { MESSAGE EXAM_RESULT FEE_REMINDER ASSIGNMENT_DUE QUIZ_SCHEDULED
                            APPROVAL_REQUEST CALENDAR_EVENT SYSTEM_ALERT LEAVE_APPROVED }
enum ClubType             { ACADEMIC SPORTS ARTS SCIENCE DRAMA MUSIC CHESS OTHER }
enum LeaveType            { SICK PERSONAL OFFICIAL MATERNITY PATERNITY }
enum LeaveStatus          { PENDING APPROVED REJECTED }
enum ApprovalStatus       { DRAFT PENDING_HOD PENDING_PROCUREMENT PENDING_ACCOUNTANT PENDING_HEAD APPROVED REJECTED ORDERED DELIVERED }
enum ResourceType         { NOTES PAST_PAPER REVISION REFERENCE VIDEO OTHER }
enum ResourceStatus       { PENDING_REVIEW APPROVED REJECTED }
enum AIReportType         { TERM_REPORT FINANCIAL_SUMMARY DEPT_SUMMARY }
enum AIReportStatus       { GENERATED REVIEWED APPROVED PUBLISHED }
enum InsightSeverity      { LOW MEDIUM HIGH }
enum TripStatus           { DRAFT PENDING_APPROVAL APPROVED COMPLETED CANCELLED }
enum ContractType         { PERMANENT CONTRACT PART_TIME }
enum AppraisalStatus      { DRAFT SELF_APPRAISED REVIEWED APPROVED }
```

---

## Index Strategy

All tables indexed on: `(school_id)` + common query patterns.

Critical indexes:
```sql
-- Attendance: daily class register
CREATE INDEX idx_attendance_class_date ON attendance_records(school_id, class_id, date);

-- Calendar: date range queries
CREATE INDEX idx_calendar_events_date ON calendar_events(school_id, start_at);

-- Analytics: pre-computed lookups
CREATE INDEX idx_analytics_entity ON analytics_summaries(school_id, entity_type, entity_id);

-- Fee payments: receipt lookup
CREATE UNIQUE INDEX idx_fee_payment_receipt ON fee_payments(mpesa_receipt_no) WHERE mpesa_receipt_no IS NOT NULL;

-- Dorm: no double-allocation
CREATE UNIQUE INDEX idx_dorm_active_allocation ON dorm_allocations(school_id, dorm_id, bed_number)
  WHERE vacated_date IS NULL;
```

---

## Migration Sequence

| Migration | Sprint | Contents |
|---|---|---|
| `0001_init_auth_tenancy` | 002 | School, User, OnboardingState, AuditLog |
| `0002_academic_core` | 003 | All Sprint 003 models |
| `0003_financial` | 004 | All Sprint 004 models |
| `0004_communications_procurement` | 005 | All Sprint 005 models |
| `0005_library` | 006 | All Sprint 006 models |
| `0006_ai` | 007 | All Sprint 007 models |
| `0007_departments` | 008 | All Sprint 008 models |

Manual SQL migrations in `prisma/migrations/manual/`:
| File | Contents |
|---|---|
| `001_rls_sprint002.sql` | RLS for Sprint 002 tables |
| `002_rls_sprint003.sql` | RLS + calendar view |
| `003_rls_sprint004.sql` | RLS + financial immutability trigger |
| `004_rls_sprint005.sql` | RLS + approval chain constraints |
| `005_rls_sprint006.sql` | RLS + storage quota trigger |
| `006_rls_sprint007.sql` | RLS for AI tables |
| `007_rls_sprint008.sql` | RLS + health visit role policy + dorm unique index |
