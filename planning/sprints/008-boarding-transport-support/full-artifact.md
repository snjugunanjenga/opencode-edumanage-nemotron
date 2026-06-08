# Sprint 008: Boarding, Transport, Nursing, Kitchen & HR — Full Artifact

**Project:** EduManage  
**Sprint:** 008  
**Status:** Approved  
**Last updated:** 2026-05-29  
**Depends on:** Sprint 007 complete  
**Significance:** Full platform complete — all 16 roles fully operational

---

# Requirements

## Sprint Goal
Complete the five remaining operational departments — Matron (boarding), Transport Manager, School Nurse, School Cateress (kitchen), and HR Manager. Each gets a full department dashboard and integrates with procurement (Sprint 005), financial management (Sprint 004), and calendar (Sprint 003/005). After this sprint, EduManage is the full product.

---

## FR-001: Boarding (School Matron)

- **Dorm management:** Create dorms (name, building, capacity, gender); Matron allocates beds to boarding students
- **Bed allocation:** One student per bed; Matron assigns on admission; transfers tracked
- **Vacancy tracking:** Real-time bed availability per dorm
- **Repair requests:** Matron raises procurement request → routes through standard procurement chain (type: REPAIR)
- **Bed purchases:** Matron raises procurement request (type: EQUIPMENT)
- **Cleaning roster:** Matron records cleaning staff assignments (text-based, no login accounts for cleaning staff)
- **Boarding fee:** Accountant configures boarding fee component in FeeStructure (Sprint 004 — `FeeCategory.BOARDING`)
- **Boarder status:** Set on student record at registration (`Student.isBoarder = true`)

**Data:**
```
Dorm { id, schoolId, name, building, totalBeds, gender (MALE|FEMALE|MIXED), isActive }
DormAllocation { id, schoolId, dormId, bedNumber, studentId, allocationDate, vacatedDate?, notes? }
CleaningRoster { id, schoolId, dormId, staffName, shift (MORNING|EVENING), weekdays Json, assignedByUserId }
```

---

## FR-002: Transport (Transport Manager)

- **Vehicle registry:** name, plate number, capacity, driver name, insurance expiry
- **Trip planning:** destination, date, departure time, vehicle, student manifest, teacher chaperones, purpose
- **Trip approval:** Transport Manager → HOD → Headteacher (uses procurement-style approval chain)
- **Trip calendar:** Approved trips auto-create CalendarEvent (eventType: TRIP) for all participants
- **Budget:** Manager submits term transport budget → Accountant reviews → Headteacher approves
- **Reports:** Trips completed, fuel costs, submitted to Accountant as PDF

**Data:**
```
Vehicle { id, schoolId, name, plateNumber, capacity, driverName, insuranceExpiry, isActive }
Trip {
  id, schoolId, vehicleId, destination, tripDate, departureTime, returnTime,
  purpose, teacherChaperones Json, status (DRAFT|PENDING_APPROVAL|APPROVED|COMPLETED|CANCELLED),
  approvalChain Json, estimatedCost, actualCost?, createdByUserId
}
TripManifest { tripId, studentId, schoolId, guardianConsent }
TransportBudget { id, schoolId, termId, submittedByUserId, amount, status, approvedByUserId? }
```

---

## FR-003: Nursing (School Nurse)

- **Health visit log:** Date, student, complaint (free text), treatment given, outcome, follow-up required flag
- **Medical inventory:** Item name, quantity, unit, reorder level, expiry date
- **Procurement integration:** Nurse raises medical supply request → standard procurement chain with category: MEDICAL
- **Health alerts:** Nurse flags a student requiring follow-up → in-app notification to Parent + Headteacher + Class Teacher
- **Confidential access:** ONLY Nurse + Headteacher can view health visit records (RLS enforced with additional role check)
- **Immunisation records:** Optional: log student immunisation history

**Data:**
```
HealthVisit {
  id, schoolId, studentId, visitDate, complaint, treatment, outcome,
  requiresFollowUp, followUpDate?, nurseUserId, isConfidential (always true)
}
MedicalInventoryItem { id, schoolId, name, quantity, unit, reorderLevel, expiryDate?, category }
ImmunisationRecord { id, schoolId, studentId, vaccineName, dateGiven, nextDueDate? }
```

**Critical:** `HealthVisit` RLS policy uses BOTH `school_id` AND additional role check — not just school_id. Implemented as Postgres policy with `current_setting('app.current_role')` check.

---

## FR-004: Kitchen (School Cateress)

- **Meal planner:** Weekly meal schedule by day and meal type (Breakfast/Lunch/Dinner/Snack)
- **Kitchen inventory:** Ingredients tracked by quantity, unit, reorder threshold
- **Auto-procurement:** When ingredient falls below reorder level → Inngest job creates draft procurement request for Cateress to review and submit
- **Budget:** Cateress submits monthly food budget → Accountant → Headteacher approval
- **Reports:** Monthly meals served + estimated cost → submitted to Accountant

**Data:**
```
MealPlan { id, schoolId, weekStartDate, createdByUserId, isPublished }
MealPlanEntry { id, mealPlanId, schoolId, dayOfWeek, mealType, menuDescription, estimatedCost }
KitchenInventoryItem { id, schoolId, name, quantity, unit, reorderLevel, lastRestockedAt }
KitchenBudget { id, schoolId, month, year, submittedByUserId, amount, status, approvedByUserId? }
```

---

## FR-005: HR Management (HR Manager)

- **Staff records:** Employment start date, contract type (PERMANENT|CONTRACT|PART_TIME), department, salary grade, emergency contact
- **Staff documents:** Upload contracts, teaching certificates, ID copies → S3; only HR + Headteacher can view
- **Leave records:** Consolidated view of all leaves (feeds from Sprint 005 `LeaveRequest`)
- **Performance appraisal:** Annual cycle — HR creates form → staff self-appraises → line manager reviews → Headteacher approves
- **Staff onboarding checklist:** New hire → HR marks off: contract signed, ID verified, email set up, added to payroll
- **Payroll integration:** HR confirms headcount each month → feeds Accountant's SalaryRun (Sprint 004)
- **Staff directory:** All staff with role, department, contact — visible to all staff roles

**Data:**
```
StaffRecord {
  id, schoolId, userId, contractType, department, salaryGrade,
  employmentStartDate, probationEndDate?, contractEndDate?,
  emergencyContactName, emergencyContactPhone, isActive
}
StaffDocument { id, schoolId, userId, documentType, s3Key, uploadedByUserId, createdAt }
PerformanceAppraisal {
  id, schoolId, userId, cycleYear, selfAppraisal?, lineManagerReview?,
  headteacherNote?, status (DRAFT|SELF_APPRAISED|REVIEWED|APPROVED), approvedAt?
}
HROnboardingChecklist {
  id, schoolId, userId, contractSigned, idVerified, emailSetUp,
  addedToPayroll, induction Completed, completedByUserId?, completedAt?
}
```

---

# Blueprint

## New Prisma Models
All entities from FR-001 through FR-005. RLS on all tables.

**Special RLS for HealthVisit:**
```sql
CREATE POLICY health_visit_confidentiality ON health_visits
  USING (
    school_id = current_setting('app.current_school_id', true)::uuid
    AND (
      current_setting('app.current_role', true) IN ('SCHOOL_NURSE', 'HEADTEACHER')
      OR current_setting('app.current_user_id', true)::uuid = nurse_user_id
    )
  );
```

## tRPC Routers

### `boarding.*`
- `createDorm`, `updateDorm`, `listDorms`
- `allocateBed({ studentId, dormId, bedNumber })` — MATRON
- `vacateBed({ allocationId })` — MATRON
- `getOccupancy()` — MATRON | HEADTEACHER
- `raiseRepairRequest({ dormId, description, items[] })` — MATRON → calls `procurement.create`

### `transport.*`
- `createVehicle`, `updateVehicle`, `listVehicles`
- `planTrip({ vehicleId, destination, tripDate, ... })` — any TEACHER | TRANSPORT_MANAGER
- `approveTrip({ tripId, step })` — appropriate approver per chain
- `confirmTripManifest({ tripId, studentIds[] })` — CLASS_TEACHER
- `submitBudget({ termId, amount })` — TRANSPORT_MANAGER
- `generateReport({ termId })` — TRANSPORT_MANAGER

### `nursing.*`
- `logVisit({ studentId, complaint, treatment, outcome, requiresFollowUp })` — SCHOOL_NURSE
- `getVisits({ studentId? })` — SCHOOL_NURSE | HEADTEACHER only
- `flagFollowUp({ visitId, followUpDate })` — SCHOOL_NURSE
- `getInventory()` + `updateInventory({ itemId, quantity })` — SCHOOL_NURSE
- `raiseProcurement({ items[] })` — SCHOOL_NURSE → calls `procurement.create`

### `kitchen.*`
- `createMealPlan({ weekStartDate })` + `addEntry({ mealPlanId, ... })` — CATERESS
- `publishMealPlan({ mealPlanId })` — CATERESS (visible to all school users)
- `getInventory()` + `updateInventory({ itemId, quantity })` — CATERESS
- `submitBudget({ month, year, amount })` — CATERESS
- `generateMonthlyReport({ month, year })` — CATERESS

### `hr.*`
- `createRecord({ userId, contractType, department, ... })` — HR_MANAGER
- `uploadDocument({ userId, documentType, s3Key })` — HR_MANAGER
- `getDocuments({ userId })` — HR_MANAGER | HEADTEACHER only
- `createAppraisal({ userId, cycleYear })` — HR_MANAGER
- `submitSelfAppraisal({ appraisalId, text })` — the appraised staff member
- `reviewAppraisal({ appraisalId, review })` — line manager
- `approveAppraisal({ appraisalId, note })` — HEADTEACHER
- `getOnboarding({ userId })` + `updateOnboarding({ userId, field, value })` — HR_MANAGER
- `confirmPayrollHeadcount({ month, year })` — HR_MANAGER → notifies Accountant

## Inngest Jobs

| Job | Trigger | Action |
|---|---|---|
| `kitchenReorderAlert` | Daily 07:00 | Check KitchenInventoryItem < reorderLevel → create draft procurement request |
| `vehicleInsuranceExpiry` | Weekly Monday | Check Vehicle.insuranceExpiry within 30 days → notify Transport Manager |
| `appraisalCycleStart` | Annual (configurable) | Create PerformanceAppraisal drafts for all active staff |
| `contractExpiryAlert` | Monthly | Check StaffRecord.contractEndDate within 60 days → notify HR Manager |

---

# Acceptance Criteria

## AC-001: Boarding
- Matron allocates Student Kamau to Dorm A, Bed 12 → `DormAllocation` created; bed no longer available for allocation
- Matron tries to allocate another student to same bed → rejected: "Bed 12 in Dorm A is already occupied"
- Matron raises dorm repair request → appears in Procurement Officer's queue with category REPAIR

## AC-002: Transport
- Teacher creates trip request → Transport Manager notified → Transport Manager approves → Headteacher notified → Headteacher approves → CalendarEvent (TRIP) created for all manifest students + chaperone teachers
- Trip at PENDING_APPROVAL status → students cannot be added to manifest yet (only after approval)

## AC-003: Nursing (Confidentiality)
- Nurse logs health visit for Student Achieng
- Teacher Kamau (CLASS_TEACHER) queries `nursing.getVisits({ studentId: achiengId })` → returns empty array (access denied at RLS level)
- Headteacher queries same → returns the visit record
- Parent queries same → returns empty array

## AC-004: Kitchen
- Ingredient "Rice" falls below reorder level (quantity: 2kg, threshold: 5kg) → Inngest creates draft procurement request for Cateress
- Cateress publishes meal plan → all staff and students see this week's menu on their calendar dashboard

## AC-005: HR
- HR Manager uploads employment contract for new teacher → `StaffDocument` created; Teacher views their own profile but cannot access their document (`hr.getDocuments` blocked by role check)
- HR Manager confirms payroll headcount for May 2026 → Accountant receives notification "HR has confirmed 45 staff for May payroll"
- Line manager submits appraisal review → Headteacher notified to approve

---

# Builder Handoff — Sprint 008

Paste into Claude Code. Sprint 007 must be passing.

**Pre-read:** all Sprint 008 planning files + planning/DOMAIN.md (all 16 roles) + planning/RISKS.md.

**Summarise before coding.** Wait for approval.

**Critical rules:**
- `HealthVisit` RLS: MUST use role-based policy (not just school_id). Test: CLASS_TEACHER querying student health visit must return 0 rows
- `StaffDocument`: HR_MANAGER + HEADTEACHER only — test this explicitly
- Trip approval chain: same state machine pattern as procurement (Sprint 005) — reuse `approvalChain Json` pattern
- Auto-procurement (kitchen): Inngest job creates DRAFT status requests only — Cateress must manually submit
- Dorm double-allocation: prevented at DB level with partial unique index: `UNIQUE (school_id, dorm_id, bed_number) WHERE vacated_date IS NULL`
- Contract expiry alerts: never send to staff member directly — only to HR Manager
- Appraisal self-submission: staff can only submit their own appraisal, not others

**Definition of done:**
All AC-001 through AC-005 pass. Health visit confidentiality test passes (CLASS_TEACHER gets empty array). Dorm double-allocation prevented at DB level. All 5 department dashboards render with correct tabs. Platform is fully operational across all 16 roles. CONTEXT.md updated: mark all 8 sprints complete. STATE.md updated: platform complete.
