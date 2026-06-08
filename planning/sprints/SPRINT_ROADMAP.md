# Sprint Roadmap — EduManage (Sprints 003–008)

**Architect:** Session 1 — 2026-05-29  
Each sprint below is a full requirements summary. Detailed blueprint + acceptance + handoff-prompt are produced by the Architect at the START of each sprint (not all upfront), because later sprints depend on decisions made during earlier implementation.

---

# Sprint 003: Academic Core

**Goal:** Schools can manage classes, timetables, attendance, exam results, and assignments — the primary daily-use features for teachers, Dean, and students.

## Scope

### Timetable & Scheduling (Dean of Academics)
- Design school timetable: assign subjects → classes → teachers → time slots → rooms
- Timetable publishes to all users' calendars
- Conflict detection: same teacher in two places, same room double-booked
- CBC and IGCSE period structures supported (different period lengths)
- Substitute teacher scheduling when leave is approved

### Attendance (Class Teacher)
- Daily digital register per class
- Mark: Present / Absent / Late / Excused
- Bulk mark (mark all present, then edit exceptions)
- Attendance summary: per student per term, per class per day
- Alert triggers (Inngest): student absent 3+ consecutive days → notify parent + Dean

### Exams & Results (Teacher → Dean → Student/Parent)
- Teacher uploads exam paper (.docx) with password confirmation
- Dean reviews + approves paper (or rejects with note)
- Teacher enters student results per exam per subject
- Dean approves results before publication
- Student/parent see results only after Dean approval
- Class performance metrics: average, highest, lowest, pass rate
- Student progress chart: score trend across terms (Recharts line chart)
- CBC grading: Exceeding / Meeting / Approaching / Below expectations
- IGCSE grading: A* through G + U

### Assignments (Teacher → Student)
- Teacher creates assignment: title, description, due date, file attachment (S3)
- Student submits: text response or file upload (S3)
- Teacher marks: score + written feedback
- Student views mark + feedback
- Assignment status: Pending / Submitted / Marked / Overdue

### Lesson Plans (Teacher)
- Teacher creates lesson plan: date, class, subject, objectives, activities, resources
- Save as draft or publish to class
- Download as .docx (Gemini-assisted formatting in Sprint 007)
- Dean can view all teachers' lesson plans

### Subject Syllabus (Dean → Teacher → Student)
- Dean uploads syllabus per subject per class (.pdf/.docx)
- All teachers and students in that class can view
- Syllabus appears on teacher's subject tab

## Key Entities (additions)
- Subject, Timetable, TimetableSlot, Room
- AttendanceRecord
- Exam, ExamResult, ExamPaper (encrypted S3 ref)
- Assignment, AssignmentSubmission
- LessonPlan, Syllabus

## Dashboard Tabs Delivered
- **Dean:** Timetable Manager, Exam Approvals, Teacher Schedules, Leave Queue
- **Class Teacher:** My Class, Attendance, My Subjects, Assignments, Exams, Lesson Plans, Calendar
- **Subject Teacher:** My Subjects, Assignments, Exams, Lesson Plans
- **Student:** My Results, My Assignments, My Timetable, Progress Chart
- **Parent:** Child's Results, Child's Attendance, Child's Timetable

---

# Sprint 004: Financial Management & Mpesa

**Goal:** Schools can collect fees via Mpesa, manage accounts, generate financial reports, handle procurement budgets, and produce KRA-compliant tax reports.

## Scope

### Fee Management (Accountant)
- School configures fee structure: fee items per class per term (tuition, activity, transport, meals, boarding)
- Student fee statement: what's owed, what's paid, balance
- Fee payment via Mpesa STK Push (parent-initiated from app)
- Fee payment via manual entry (cash/bank — accountant records)
- Mpesa C2B paybill integration (school's own paybill number)
- Auto-reconciliation: Mpesa receipts matched to student fee records
- Fee reminders: Inngest scheduled job, sends email (Resend) + in-app notification 7 days before term end
- Mpesa statement download: PDF + CSV, filtered by date/type/student

### School Subscription Billing
- Stripe Checkout for new school subscriptions
- Mpesa for local school subscription renewal
- Subscription status visible to Headteacher + Super Admin
- Trial → Active on first payment
- Suspended after 14 days unpaid (Inngest job)

### Financial Reports (Accountant + Headteacher)
- Term income report: total fees collected, outstanding, breakdown by class
- Expenditure report: procurement + salary total per term
- Cash flow statement: inflows vs outflows per month
- Budget vs actual per department
- All reports downloadable as PDF + Excel

### Payroll (Accountant + Headteacher approval)
- Staff salary records: basic pay, allowances, deductions
- PAYE calculation (Kenya 2024 tax bands)
- Net pay calculation
- Payslip generation (PDF, Resend to staff email)
- Salary payment: Mpesa B2C (Premium tier) or manual flag
- Headteacher approves salary run before disbursement

### KRA Tax Reports
- PAYE monthly return: formatted for iTax manual upload (CSV)
- VAT return: if school is VAT-registered (16% on applicable services)
- Withholding Tax: 5% on supplier payments above threshold
- Reports clearly marked "for review by tax professional"

### Bank Statement Import
- Accountant uploads bank statement (CSV/Excel)
- System parses + maps to internal transactions
- Unmatched items flagged for manual matching

## Key Entities (additions)
- FeeStructure, FeeItem, StudentFeeBalance
- FeePayment (Mpesa + manual)
- Subscription, StripeSubscription, MpesaSubscription
- SalaryRecord, Payslip, SalaryRun
- TaxReturn (PAYE, VAT, WHT)
- BankStatement, BankTransaction

## Dashboard Tabs Delivered
- **Accountant:** Dashboard, Payments, School Fees, Subscriptions, Reports, Budgets, Accounts, Tax, Calendar, Messages
- **Headteacher:** Financial Overview (read-only), Salary Approval, Budget Approvals
- **Parent:** Fee Statement, Pay Now (Mpesa STK), Download Receipts, Download Mpesa Statement

---

# Sprint 005: Communications & Calendar

**Goal:** All school stakeholders can communicate securely, and every user has a unified calendar showing their personal schedule, school events, club events, and class events.

## Scope

### Messaging System
- Direct messages: 1-to-1 between any two users in same school
- Group messages: pre-defined groups (All Staff, All Parents, Grade 5 Parents, Class 7A)
- Custom group creation by Headteacher / Dean / Class Teacher
- Group send confirmation: "You are about to message N people — confirm?"
- Message history stored + searchable
- File attachments in messages (S3, 10MB limit)
- In-app notification badge for unread messages
- Email notification (Resend) for new message (configurable per user)

### Notifications System
- Notification types: message, exam result, fee reminder, assignment due, approval request, calendar event, system alert
- In-app notification centre (bell icon)
- Per-user notification preferences (which types trigger email)
- Bulk notification to role group (Headteacher broadcast)

### Calendar & Planner
- Personal calendar per user: auto-populated with timetable, assignments, exams
- School calendar: events created by Admin/Dean, visible to all
- Class calendar: events created by Class Teacher, visible to class members
- Club calendar: events created by club patron, visible to members
- Trip planning: teacher creates trip → Transport Manager confirms → HOD/Headteacher approves → appears on all involved calendars
- Event types: Academic, Sports, Trip, Holiday, Meeting, Exam, Club
- Export calendar to .ics (Google Calendar / Apple Calendar compatible)

### Clubs & Groups
- Student joins/leaves clubs (requires patron approval for restricted clubs)
- Teacher assigned as patron of one or more clubs
- Club has: name, description, members, patron, schedule, events, progress notes
- Club progress log: patron records activities and achievements
- Club budget request → Accountant → Headteacher

### Leave Management (Teachers)
- Teacher submits leave request: dates, type (sick/personal/official), reason
- Dean reviews: approve or reject with note
- If approved: Dean assigns substitute teacher for affected classes
- Substitute assignment updates timetable + notifies substitute teacher
- Leave calendar visible to Dean + Headteacher

## Key Entities (additions)
- Message, MessageThread, MessageRecipient
- Notification, NotificationPreference
- CalendarEvent, EventAttendee
- Club, ClubMembership, ClubEvent, ClubBudgetRequest
- LeaveRequest, SubstituteAssignment

## Dashboard Tabs Delivered (additions to existing dashboards)
- **All roles:** Messages tab, Notifications centre, Calendar tab
- **Dean:** Leave Queue, Substitute Scheduler
- **Headteacher:** Broadcast Messages, Approval Queue (trips, club budgets)
- **Student:** Clubs tab, My Calendar, Messages

---

# Sprint 006: Library & Learning Resources

**Goal:** Students have a structured digital library for each subject; teachers manage and upload curriculum resources; school librarian oversees the system; parents can monitor their child's library activity.

## Scope

### School Library (Librarian-managed)
- Resource upload: .docx, .pdf, links, videos (YouTube embed)
- Tag by: subject, class, curriculum type (CBC/IGCSE), grade level, resource type
- Librarian approves teacher-uploaded resources before publication
- Librarian monitors student upload/download activity
- Storage quota enforcement: tier-based school-wide quota (Basic 50GB / Standard 200GB / Premium 1TB)
- Storage usage dashboard for Librarian + IT Support

### Teacher Subject Resources
- Teacher uploads resources to their subjects (visible to their assigned classes only)
- Resource types: lesson notes, past papers, revision materials, reference links
- Resources linked to syllabus topics

### Student Personal Library
- Student uploads/saves: own notes (.docx/.pdf), bookmarked links
- Per-student storage quota (operator to confirm via Q-002; default 500MB)
- Search across own library + school library
- Resources viewable offline (PWA cache — v2)

### Syllabus View (Dean → Teachers → Students)
- Dean uploads syllabus per subject per class
- Syllabus broken into topics
- Teacher marks topic as "taught" (progress tracking)
- Student views syllabus with teacher's progress markers

### Parent Library Monitoring
- Parent views child's library activity: what was accessed, when, uploads
- Parent can suggest resources to child (sends as in-app recommendation)

## Key Entities (additions)
- LibraryResource, ResourceTag
- StudentLibraryItem, StudentLibraryActivity
- SyllabusDocument, SyllabusTopic, SyllabusProgress
- StorageQuota, StorageUsage

## Dashboard Tabs Delivered
- **Librarian:** Full dashboard — Resources, Upload Queue, Student Activity, Storage, Syllabus
- **Teacher:** Subject Resources tab, Upload, Syllabus view
- **Student:** Library tab (school resources + personal), Syllabus
- **Parent:** Child's Library Activity

---

# Sprint 007: AI Automation Layer

**Goal:** Premium-tier schools get AI-powered insights, automated report generation, anomaly detection, and smart suggestions — powered by Google Gemini 1.5 Pro via async Inngest jobs.

## Scope

### AI Report Generation
- **Narrative Term Report** (Dean/Headteacher): Gemini generates a prose performance summary for each student based on exam results, attendance, assignment completion. Headteacher reviews + edits before publishing.
- **Financial Summary Narrative** (Accountant/Headteacher): Gemini narrates the term's financial performance from raw data.
- **Department Reports**: Procurement spend analysis, library usage patterns.

### Anomaly Detection (Inngest scheduled — runs nightly)
- Attendance anomalies: students with >20% absence rate flagged → notification to Class Teacher + Dean
- Fee defaulters: students with >50% outstanding balance at week 6 of term → notification to Accountant + Headteacher
- Exam score drops: student drops >20% from previous term → flag to Class Teacher + Dean
- Procurement overspend: department exceeds budget by >10% → alert to Accountant

### Timetable Conflict Detection
- On every timetable save: Gemini (or deterministic logic) checks for:
  - Same teacher in two classes at same time
  - Same room double-booked
  - Class has >2 consecutive periods of same subject
- Conflicts flagged immediately in Dean's UI with suggested resolutions

### Smart Suggestions
- Assignment feedback suggestions: Teacher enters raw mark → Gemini suggests constructive written feedback
- Procurement suggestions: Based on historical procurement patterns, suggest reorder items before they run out
- Lesson plan assistant: Teacher describes lesson topic → Gemini generates structured lesson plan draft

### AI Usage Controls
- All AI features: Premium tier only
- Usage tracked per school (tokens consumed)
- Rate limit: 100 AI requests/day per school on Standard (if unlocked), unlimited on Premium
- All AI outputs clearly labelled "AI-generated — please review before publishing"

## Key Entities (additions)
- AIReport, AIInsight, AIUsageLog

## Dashboard Additions
- **Headteacher:** AI Insights panel on dashboard home
- **Dean:** AI Timetable Checker, AI Report Generator
- **Accountant:** AI Financial Insights
- **Class Teacher:** AI Feedback Suggestions on assignment marking

---

# Sprint 008: Boarding, Transport, Nursing, Kitchen & HR

**Goal:** Complete the remaining operational departments — School Matron (boarding), Transport Manager, School Nurse, School Cateress (kitchen), and HR Manager — each with their own dashboard, workflows, and procurement integration.

## Scope

### Boarding (School Matron)
- Student boarding registration flag
- Dorm management: create dorms, set capacity, allocate students to dorm + bed
- Bed allocation: one student per bed; vacancy tracking
- Dorm repair requests → procurement workflow
- Additional bed purchase → procurement workflow
- Cleaning staff roster (managed by Matron, no platform login required)
- Boarding fee integration: Accountant configures boarding fee component

### Transport (Transport Manager)
- Vehicle registry: name, plate, capacity, driver
- Trip planning: create trip (destination, date, vehicle, student manifest, teacher chaperones)
- Trip approval workflow: Transport Manager → relevant HOD → Headteacher
- Trip calendar: approved trips visible on school calendar
- Transport budget: Manager submits term budget → Accountant → Headteacher
- Transport report: trips completed, costs, submitted to Accountant

### Nursing (School Nurse)
- Student health visit log: date, student, complaint, treatment, outcome
- Medical inventory: medicines + supplies tracked
- Medical supply procurement: Nurse raises request → Procurement Officer
- Health alerts: Nurse flags a student requiring follow-up → notifies parents + Headteacher
- Confidential records: only Nurse + Headteacher can view student health records

### Kitchen (School Cateress)
- Meal planner: weekly meal schedule per day
- Kitchen inventory: ingredients, quantities, reorder thresholds
- Auto-procurement trigger: item below reorder threshold → Cateress raises procurement request
- Kitchen budget: Cateress submits term food budget → Accountant → Headteacher
- Kitchen report: meals served, costs, submitted monthly to Accountant

### HR Management (HR Manager)
- Staff records: employment start date, contract type, department, salary grade
- Staff documents: upload contracts, certificates (S3) — only HR + Headteacher can view
- Leave records: consolidated view of all approved/pending leaves (feeds from Sprint 005)
- Performance appraisal: annual cycle, HR initiates, Headteacher approves
- Staff onboarding checklist: new hire → HR completes document collection → marks complete
- Payroll integration: HR confirms headcount → feeds into Accountant's salary run (Sprint 004)

### Procurement Integration (across all new departments)
All 5 departments raise procurement requests through the unified procurement workflow from Sprint 003. No new procurement logic — just new requestor roles and category tags.

## Key Entities (additions)
- Dorm, DormAllocation, DormRepairRequest
- Vehicle, Trip, TripManifest, TripApproval
- HealthVisit, MedicalInventory
- MealPlan, KitchenInventory, KitchenBudget
- StaffRecord, StaffDocument, PerformanceAppraisal, HRChecklist

## Dashboard Tabs Delivered
- **Matron:** Dashboard, Dorms, Allocations, Repairs, Procurement, Reports, Messages, Calendar
- **Transport Manager:** Dashboard, Vehicles, Trips, Budget, Reports, Messages, Calendar
- **School Nurse:** Dashboard, Health Visits, Medical Inventory, Alerts, Procurement, Messages, Calendar
- **Cateress:** Dashboard, Meal Planner, Kitchen Inventory, Budget, Reports, Procurement, Messages, Calendar
- **HR Manager:** Dashboard, Staff Records, Documents, Leave Overview, Appraisals, Onboarding, Messages, Calendar
- **Headteacher:** New approval tabs for boarding/transport/nursing/kitchen/HR reports

---

## Sprint Delivery Summary

| Sprint | Chargeable? | Unlocks |
|---|---|---|
| 002 | No | Foundation for everything |
| 003 | No | Core academic value |
| 004 | **YES** | First billable product (Ksh 50,000/term) |
| 005 | **YES** | Standard tier feature |
| 006 | **YES** | Standard tier feature |
| 007 | **YES** (Premium only) | Premium tier differentiator |
| 008 | **YES** | Full platform — complete product |

**First revenue target:** 15 schools onboarded at Sprint 004 completion → Ksh 750,000/term.
