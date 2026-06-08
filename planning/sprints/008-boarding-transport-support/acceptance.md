# Acceptance Criteria — Sprint 008: Boarding, Transport, Nursing, Kitchen & HR

**Last updated:** 2026-05-29

---

## AC-001: Boarding — Dorm Allocation & Double-Booking Prevention

**Given** Matron allocates Student Kamau to Dorm A, Bed 7  
**When** `boarding.allocateBed` is called  
**Then** `DormAllocation` created with `vacatedDate = null`; Dorm A shows Bed 7 as occupied in occupancy view

**Given** Matron tries to allocate Student Achieng to the same Dorm A, Bed 7  
**When** `boarding.allocateBed` called  
**Then** tRPC returns `BAD_REQUEST: "Bed 7 in Dorm A is already occupied"` — DB unique partial index prevents insert even if application check is bypassed

**Given** Matron vacates Student Kamau from Bed 7 (sets `vacatedDate = today`)  
**When** Matron then allocates Student Achieng to Dorm A, Bed 7  
**Then** new `DormAllocation` created successfully — partial index only blocks WHERE `vacatedDate IS NULL`

**Given** Matron raises a repair request for Dorm B leaking roof  
**When** `boarding.raiseProcurement` called  
**Then** `ProcurementRequest` created with `department = BOARDING`, `category = REPAIR`, `status = DRAFT`; routes through standard procurement approval chain

---

## AC-002: Transport — Trip Approval & Calendar

**Given** Teacher Wanjiku plans a trip to Nairobi National Museum on 15 July  
**When** trip submitted for approval  
**Then** `Trip.status = PENDING_APPROVAL`; Transport Manager notified

**Given** Transport Manager approves → HOD approves → Headteacher approves  
**When** final approval given  
**Then** `Trip.status = APPROVED`; `CalendarEvent` rows created (type: TRIP) for all students in manifest + chaperone teachers; all affected users see trip on their calendar

**Given** vehicle has insurance expiring in 20 days  
**When** weekly Inngest `vehicleInsuranceExpiry` job runs  
**Then** Transport Manager + Headteacher receive in-app notification: "Vehicle [name] insurance expires on [date]"

**Given** Headteacher rejects the trip  
**When** rejected with note "Not enough parental consents received"  
**Then** `Trip.status = CANCELLED`; Teacher Wanjiku notified with rejection note; no CalendarEvent created

---

## AC-003: Nursing — Confidentiality (Critical)

**Given** Nurse logs a health visit for Student Kamau  
**When** `health_visits` row inserted  
**Then** record exists in DB with `schoolId` and `nurseUserId`

**Given** Class Teacher Odhiambo (CLASS_TEACHER role) calls `nursing.getVisits({ studentId: kamauId })`  
**When** tRPC procedure runs  
**Then** returns `FORBIDDEN` at tRPC layer (role not in allowedRoles) — record never reaches DB query

**Given** even if tRPC check somehow bypassed and direct DB query runs with CLASS_TEACHER role context  
**When** query runs against `health_visits` table  
**Then** RLS policy returns 0 rows (role `CLASS_TEACHER` not in `('SCHOOL_NURSE', 'HEADTEACHER')`)

**Given** Parent of Student Kamau  
**When** parent calls any nursing endpoint  
**Then** returns `FORBIDDEN` — parent role blocked entirely from nursing router

**Given** Nurse sets `requiresFollowUp = true` for Kamau's visit  
**When** visit saved  
**Then** Parent of Kamau + Headteacher + Kamau's Class Teacher each receive in-app notification: "Student Kamau requires a follow-up health check" — notification contains NO medical details, only the alert

---

## AC-004: Kitchen — Auto-Procurement & Meal Plan

**Given** Rice stock falls to 2kg, reorder level is 5kg  
**When** daily Inngest `kitchenReorderCheck` job runs at 07:00  
**Then** `ProcurementRequest` created with `status = DRAFT`, `category = FOOD`, `department = KITCHEN`; Cateress receives notification: "1 item below reorder level — review in Procurement"

**Given** Cateress has not submitted the draft request  
**When** job runs again the next day (Rice still at 2kg)  
**Then** NO duplicate draft created — idempotent check: only create if no existing DRAFT/PENDING request for same item in current week

**Given** Cateress publishes the week's meal plan  
**When** `kitchen.publishMealPlan` called  
**Then** `MealPlan.isPublished = true`; all staff and students can see "This Week's Menu" on their calendar dashboard; `CalendarEvent` of type SCHOOL created for each day with meal info in description

---

## AC-005: HR — Staff Documents & Appraisal

**Given** HR Manager uploads Teacher Kamau's employment contract  
**When** `hr.uploadDocument` called with `documentType = CONTRACT`  
**Then** `StaffDocument` row created; S3 key stored

**Given** Teacher Kamau (CLASS_TEACHER role) calls `hr.getDocuments({ userId: kamauId })`  
**When** tRPC runs  
**Then** returns `FORBIDDEN` — CLASS_TEACHER not in allowedRoles for `hr.getDocuments`

**Given** HR Manager confirms payroll headcount for June 2026  
**When** `hr.confirmPayrollHeadcount({ month: 6, year: 2026 })` called  
**Then** Accountant receives notification: "HR has confirmed 45 active staff for June 2026 payroll. You may now run the salary."

**Given** HR Manager creates annual appraisal for Teacher Wanjiku  
**When** created  
**Then** `PerformanceAppraisal.status = DRAFT`; Teacher Wanjiku notified to complete self-appraisal

**Given** Teacher Wanjiku submits self-appraisal text  
**When** `hr.submitSelfAppraisal` called  
**Then** `status = SELF_APPRAISED`; line manager notified to review

**Given** Teacher Wanjiku attempts to submit self-appraisal for a colleague  
**When** `hr.submitSelfAppraisal({ appraisalId: colleagueAppraisalId })` called  
**Then** tRPC returns `FORBIDDEN: "You can only submit your own self-appraisal"`

---

## AC-006: prismaWithRLS Role Context (Breaking Change)

**Given** the updated `prismaWithRLS(schoolId, userId, role)` signature  
**When** any tRPC procedure calls it  
**Then** `app.current_role` is correctly set in the Postgres session; confirmed by:
- Nurse query returns health visits ✅
- Class Teacher query returns 0 health visits ✅
- HR Manager query returns staff documents ✅
- Class Teacher query returns 0 staff documents ✅

---

## Regression Tests (Full Platform)

After Sprint 008, run the complete regression suite:

- Sprint 002: auth, RLS tenant isolation, all 16 roles route correctly
- Sprint 003: attendance, quizzes, exams, analytics, calendar
- Sprint 004: Mpesa idempotency, PAYE calculation, Stripe webhook
- Sprint 005: group message safety gate, procurement state machine
- Sprint 006: library quota enforcement, parent access control
- Sprint 007: Premium gate, AI banner on all AI content, token logging
- Sprint 008: health visit confidentiality (critical), dorm double-booking prevention

---

## Definition of Done

All AC-001 through AC-006 pass.  
**Health visit confidentiality: BOTH tRPC AND RLS layers tested — non-negotiable.**  
Dorm double-allocation: prevented by DB partial unique index (tested by attempting concurrent inserts).  
prismaWithRLS role context: verified with integration tests for SCHOOL_NURSE and HR_MANAGER role policies.  
All 5 Inngest jobs tested with `inngest-cli dev`.  
All new tables RLS-protected.  
Full platform regression suite passing.  
`CONTEXT.md` Sprint History: all 8 sprints marked complete.  
`STATE.md`: platform complete — ready for first school onboarding.
