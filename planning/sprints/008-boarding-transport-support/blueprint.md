# Blueprint — Sprint 008: Boarding, Transport, Nursing, Kitchen & HR

**Project:** EduManage  
**Sprint:** 008  
**Last updated:** 2026-05-29

---

## New Prisma Models

```prisma
// ─── Boarding ────────────────────────────────────────────────────

model Dorm {
  id          String   @id @default(uuid())
  schoolId    String
  name        String
  building    String?
  totalBeds   Int
  gender      DormGender @default(MIXED)
  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())
  allocations DormAllocation[]
  @@index([schoolId])
}

model DormAllocation {
  id             String    @id @default(uuid())
  schoolId       String
  dormId         String
  bedNumber      Int
  studentId      String
  allocationDate DateTime  @db.Date @default(now())
  vacatedDate    DateTime? @db.Date
  notes          String?
  dorm           Dorm      @relation(fields: [dormId], references: [id])
  @@index([schoolId, studentId])
  // Prevent double-allocation: unique active bed
  // Applied as manual SQL partial unique index:
  // CREATE UNIQUE INDEX dorm_active_bed ON dorm_allocations(school_id, dorm_id, bed_number)
  //   WHERE vacated_date IS NULL;
}

model CleaningRoster {
  id              String   @id @default(uuid())
  schoolId        String
  dormId          String
  staffName       String
  shift           CleanShift
  weekdays        Json     // [1,2,3,4,5] — 1=Mon
  assignedByUserId String
  createdAt       DateTime @default(now())
  @@index([schoolId, dormId])
}

enum DormGender { MALE FEMALE MIXED }
enum CleanShift { MORNING EVENING }

// ─── Transport ───────────────────────────────────────────────────

model Vehicle {
  id               String   @id @default(uuid())
  schoolId         String
  name             String
  plateNumber      String
  capacity         Int
  driverName       String
  insuranceExpiry  DateTime @db.Date
  isActive         Boolean  @default(true)
  createdAt        DateTime @default(now())
  trips            Trip[]
  @@index([schoolId])
}

model Trip {
  id               String     @id @default(uuid())
  schoolId         String
  vehicleId        String
  destination      String
  tripDate         DateTime   @db.Date
  departureTime    String     // "08:00"
  returnTime       String?    // "17:00"
  purpose          String
  teacherChaperones Json      // String[] — userId array
  status           TripStatus @default(DRAFT)
  approvalChain    Json       @default("[]")
  estimatedCost    Decimal?   @db.Decimal(12,2)
  actualCost       Decimal?   @db.Decimal(12,2)
  createdByUserId  String
  createdAt        DateTime   @default(now())
  manifest         TripManifest[]
  vehicle          Vehicle    @relation(fields: [vehicleId], references: [id])
  @@index([schoolId, tripDate])
}

model TripManifest {
  tripId           String
  studentId        String
  schoolId         String
  guardianConsent  Boolean  @default(false)
  trip             Trip     @relation(fields: [tripId], references: [id])
  @@id([tripId, studentId])
  @@index([schoolId])
}

model TransportBudget {
  id              String   @id @default(uuid())
  schoolId        String
  termId          String
  submittedByUserId String
  amount          Decimal  @db.Decimal(12,2)
  status          String   @default("PENDING")
  approvedByUserId String?
  approvedAt      DateTime?
  createdAt       DateTime @default(now())
  @@unique([schoolId, termId])
}

enum TripStatus { DRAFT PENDING_APPROVAL APPROVED COMPLETED CANCELLED }

// ─── Nursing ─────────────────────────────────────────────────────

model HealthVisit {
  id               String   @id @default(uuid())
  schoolId         String
  studentId        String
  visitDate        DateTime @db.Date
  complaint        String
  treatment        String
  outcome          String
  requiresFollowUp Boolean  @default(false)
  followUpDate     DateTime? @db.Date
  nurseUserId      String
  createdAt        DateTime @default(now())
  // CONFIDENTIAL: only SCHOOL_NURSE + HEADTEACHER — enforced in RLS (see manual SQL)
  @@index([schoolId, studentId])
}

model MedicalInventoryItem {
  id           String   @id @default(uuid())
  schoolId     String
  name         String
  quantity     Decimal  @db.Decimal(10,2)
  unit         String
  reorderLevel Decimal  @db.Decimal(10,2)
  expiryDate   DateTime? @db.Date
  category     String   @default("GENERAL")
  @@index([schoolId])
}

model ImmunisationRecord {
  id          String   @id @default(uuid())
  schoolId    String
  studentId   String
  vaccineName String
  dateGiven   DateTime @db.Date
  nextDueDate DateTime? @db.Date
  recordedByNurseId String
  @@index([schoolId, studentId])
}

// ─── Kitchen ─────────────────────────────────────────────────────

model MealPlan {
  id              String   @id @default(uuid())
  schoolId        String
  weekStartDate   DateTime @db.Date
  createdByUserId String
  isPublished     Boolean  @default(false)
  publishedAt     DateTime?
  createdAt       DateTime @default(now())
  entries         MealPlanEntry[]
  @@unique([schoolId, weekStartDate])
  @@index([schoolId])
}

model MealPlanEntry {
  id               String   @id @default(uuid())
  mealPlanId       String
  schoolId         String
  dayOfWeek        Int      // 1=Mon … 5=Fri + 6=Sat, 7=Sun for boarding
  mealType         MealType
  menuDescription  String
  estimatedCost    Decimal? @db.Decimal(10,2)
  plan             MealPlan @relation(fields: [mealPlanId], references: [id])
  @@unique([mealPlanId, dayOfWeek, mealType])
  @@index([schoolId])
}

model KitchenInventoryItem {
  id              String   @id @default(uuid())
  schoolId        String
  name            String
  quantity        Decimal  @db.Decimal(10,2)
  unit            String
  reorderLevel    Decimal  @db.Decimal(10,2)
  lastRestockedAt DateTime?
  updatedByUserId String
  updatedAt       DateTime @updatedAt
  @@index([schoolId])
}

model KitchenBudget {
  id               String   @id @default(uuid())
  schoolId         String
  month            Int
  year             Int
  submittedByUserId String
  amount           Decimal  @db.Decimal(12,2)
  status           String   @default("PENDING")
  approvedByUserId String?
  approvedAt       DateTime?
  createdAt        DateTime @default(now())
  @@unique([schoolId, month, year])
}

enum MealType { BREAKFAST LUNCH DINNER SNACK }

// ─── HR ──────────────────────────────────────────────────────────

model StaffRecord {
  id                    String       @id @default(uuid())
  schoolId              String
  userId                String       @unique
  contractType          ContractType
  department            String
  salaryGrade           String?
  employmentStartDate   DateTime     @db.Date
  probationEndDate      DateTime?    @db.Date
  contractEndDate       DateTime?    @db.Date
  emergencyContactName  String?
  emergencyContactPhone String?
  isActive              Boolean      @default(true)
  createdAt             DateTime     @default(now())
  documents             StaffDocument[]
  appraisals            PerformanceAppraisal[]
  onboarding            HROnboardingChecklist?
  @@index([schoolId])
}

model StaffDocument {
  id               String   @id @default(uuid())
  schoolId         String
  userId           String
  documentType     String   // CONTRACT | CERTIFICATE | ID | OTHER
  s3Key            String
  uploadedByUserId String
  createdAt        DateTime @default(now())
  record           StaffRecord @relation(fields: [userId], references: [userId])
  // Access: HR_MANAGER + HEADTEACHER only — enforced at tRPC + RLS
  @@index([schoolId, userId])
}

model PerformanceAppraisal {
  id                 String          @id @default(uuid())
  schoolId           String
  userId             String
  cycleYear          Int
  selfAppraisal      String?         @db.Text
  lineManagerReview  String?         @db.Text
  lineManagerUserId  String?
  headteacherNote    String?         @db.Text
  status             AppraisalStatus @default(DRAFT)
  createdAt          DateTime        @default(now())
  approvedAt         DateTime?
  record             StaffRecord     @relation(fields: [userId], references: [userId])
  @@unique([schoolId, userId, cycleYear])
  @@index([schoolId])
}

model HROnboardingChecklist {
  id                  String    @id @default(uuid())
  schoolId            String
  userId              String    @unique
  contractSigned      Boolean   @default(false)
  idVerified          Boolean   @default(false)
  emailSetUp          Boolean   @default(false)
  addedToPayroll      Boolean   @default(false)
  inductionCompleted  Boolean   @default(false)
  completedByUserId   String?
  completedAt         DateTime?
  record              StaffRecord @relation(fields: [userId], references: [userId])
  @@index([schoolId])
}

enum ContractType    { PERMANENT CONTRACT PART_TIME }
enum AppraisalStatus { DRAFT SELF_APPRAISED REVIEWED APPROVED }
```

---

## tRPC Routers

### `boarding.*`
```typescript
boarding.createDorm({ name, building, totalBeds, gender })             // MATRON | HT
boarding.updateDorm({ dormId, ...fields })                              // MATRON | HT
boarding.listDorms()                                                    // MATRON | HT | HEADTEACHER
boarding.getOccupancy({ dormId? })                                      // MATRON | HT
boarding.allocateBed({ studentId, dormId, bedNumber, notes? })          // MATRON
boarding.vacateBed({ allocationId, vacatedDate })                       // MATRON
boarding.transferStudent({ studentId, newDormId, newBedNumber })        // MATRON — vacates + reallocates
boarding.updateCleaningRoster({ dormId, entries[] })                    // MATRON
boarding.raiseProcurement({ dormId, description, items[] })             // MATRON — calls procurement.create
```

### `transport.*`
```typescript
transport.createVehicle({ name, plateNumber, capacity, driverName, insuranceExpiry }) // TRANSPORT_MANAGER
transport.updateVehicle({ vehicleId, ...fields })                                      // TRANSPORT_MANAGER
transport.listVehicles()                                                               // TRANSPORT_MANAGER | HT
transport.planTrip({ vehicleId, destination, tripDate, ... })                          // ANY TEACHER | TRANSPORT_MANAGER
transport.updateManifest({ tripId, studentIds[], guardianConsent? })                   // CLASS_TEACHER
transport.approveTrip({ tripId, note? })                                               // current approver
transport.rejectTrip({ tripId, note })                                                 // any approver
transport.completeTrip({ tripId, actualCost? })                                        // TRANSPORT_MANAGER
transport.submitBudget({ termId, amount })                                             // TRANSPORT_MANAGER
transport.approveBudget({ budgetId })                                                  // ACCOUNTANT | HT
transport.generateReport({ termId })                                                   // TRANSPORT_MANAGER
```

### `nursing.*`
```typescript
nursing.logVisit({ studentId, visitDate, complaint, treatment, outcome, requiresFollowUp, followUpDate? })
  // SCHOOL_NURSE only
nursing.getVisits({ studentId? })
  // SCHOOL_NURSE | HEADTEACHER ONLY — tRPC role check + RLS
nursing.flagFollowUp({ visitId, followUpDate })                      // SCHOOL_NURSE
nursing.getInventory()                                               // SCHOOL_NURSE
nursing.updateInventory({ itemId, quantity })                        // SCHOOL_NURSE
nursing.raiseProcurement({ items[] })                               // SCHOOL_NURSE
nursing.logImmunisation({ studentId, vaccineName, dateGiven, nextDueDate? }) // SCHOOL_NURSE
nursing.getImmunisations({ studentId })                             // SCHOOL_NURSE | HT
```

### `kitchen.*`
```typescript
kitchen.createMealPlan({ weekStartDate })                            // CATERESS
kitchen.addEntry({ mealPlanId, dayOfWeek, mealType, menuDescription, estimatedCost? }) // CATERESS
kitchen.publishMealPlan({ mealPlanId })                             // CATERESS → creates CalendarEvents
kitchen.getInventory()                                              // CATERESS
kitchen.updateInventory({ itemId, quantity })                       // CATERESS
kitchen.submitBudget({ month, year, amount })                      // CATERESS
kitchen.approveBudget({ budgetId })                                // ACCOUNTANT | HT
kitchen.generateMonthlyReport({ month, year })                     // CATERESS
```

### `hr.*`
```typescript
hr.createRecord({ userId, contractType, department, employmentStartDate, ... }) // HR_MANAGER
hr.updateRecord({ recordId, ...fields })                                          // HR_MANAGER
hr.getRecord({ userId? })                                                         // HR_MANAGER | HT | STAFF(own basic info)
hr.uploadDocument({ userId, documentType, s3Key })                               // HR_MANAGER
hr.getDocuments({ userId })                                                       // HR_MANAGER | HT ONLY
hr.createAppraisal({ userId, cycleYear })                                        // HR_MANAGER
hr.submitSelfAppraisal({ appraisalId, text })                                    // appraisee (own only)
hr.submitLineManagerReview({ appraisalId, review })                              // line manager
hr.approveAppraisal({ appraisalId, note? })                                      // HT
hr.getOnboarding({ userId })                                                     // HR_MANAGER
hr.updateOnboarding({ userId, field, value })                                    // HR_MANAGER
hr.confirmPayrollHeadcount({ month, year })                                      // HR_MANAGER → notifies Accountant
hr.getStaffDirectory()                                                           // ALL STAFF (name, role, dept, email)
hr.getLeaveOverview({ termId? })                                                 // HR_MANAGER | HT
```

---

## Inngest Jobs (Sprint 008)

```typescript
// inngest/functions/vehicle-insurance-expiry.ts
// Cron: weekly Monday 08:00
// Check Vehicle.insuranceExpiry <= today + 30 days
// Notify TRANSPORT_MANAGER + HEADTEACHER per school

// inngest/functions/contract-expiry-alert.ts
// Cron: 1st of each month 08:00
// Check StaffRecord.contractEndDate <= today + 60 days
// Notify HR_MANAGER (NOT the staff member directly)

// inngest/functions/kitchen-reorder-check.ts
// Cron: daily 07:00
// Check KitchenInventoryItem.quantity <= reorderLevel
// For each low-stock item: create DRAFT ProcurementRequest (category: FOOD)
// Notify CATERESS: "3 items below reorder level — review and submit procurement"

// inngest/functions/appraisal-cycle-start.ts
// Trigger: hr/appraisal-cycle.started (manual trigger by HR_MANAGER)
// Creates PerformanceAppraisal DRAFT rows for all active staff
// Notifies each staff member to complete self-appraisal

// inngest/functions/trip-approved-calendar.ts
// Trigger: transport/trip.approved
// Creates CalendarEvent (TRIP) for each student in TripManifest
// Creates CalendarEvent for each chaperone teacher
```

---

## Special RLS: Health Visit Confidentiality

```sql
-- prisma/migrations/manual/007_rls_sprint008.sql

-- Standard tenant isolation for all Sprint 008 tables
ALTER TABLE dorms ENABLE ROW LEVEL SECURITY;
ALTER TABLE dorm_allocations ENABLE ROW LEVEL SECURITY;
ALTER TABLE cleaning_rosters ENABLE ROW LEVEL SECURITY;
ALTER TABLE vehicles ENABLE ROW LEVEL SECURITY;
ALTER TABLE trips ENABLE ROW LEVEL SECURITY;
ALTER TABLE trip_manifests ENABLE ROW LEVEL SECURITY;
ALTER TABLE transport_budgets ENABLE ROW LEVEL SECURITY;
ALTER TABLE health_visits ENABLE ROW LEVEL SECURITY;
ALTER TABLE medical_inventory_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE immunisation_records ENABLE ROW LEVEL SECURITY;
ALTER TABLE meal_plans ENABLE ROW LEVEL SECURITY;
ALTER TABLE meal_plan_entries ENABLE ROW LEVEL SECURITY;
ALTER TABLE kitchen_inventory_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE kitchen_budgets ENABLE ROW LEVEL SECURITY;
ALTER TABLE staff_records ENABLE ROW LEVEL SECURITY;
ALTER TABLE staff_documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE performance_appraisals ENABLE ROW LEVEL SECURITY;
ALTER TABLE hr_onboarding_checklists ENABLE ROW LEVEL SECURITY;

-- Standard tenant isolation for most tables
CREATE POLICY tenant_isolation ON dorms
  USING (school_id = current_setting('app.current_school_id', true)::uuid);
-- (repeat for all tables EXCEPT health_visits and staff_documents)

-- HEALTH VISIT: confidential — role-based policy
CREATE POLICY health_visit_confidential ON health_visits
  USING (
    school_id = current_setting('app.current_school_id', true)::uuid
    AND (
      current_setting('app.current_role', true) IN ('SCHOOL_NURSE', 'HEADTEACHER')
    )
  );

-- STAFF DOCUMENTS: HR + Headteacher only
CREATE POLICY staff_documents_restricted ON staff_documents
  USING (
    school_id = current_setting('app.current_school_id', true)::uuid
    AND (
      current_setting('app.current_role', true) IN ('HR_MANAGER', 'HEADTEACHER')
    )
  );

-- DORM ALLOCATION: partial unique index (prevents double-booking)
CREATE UNIQUE INDEX dorm_active_bed_unique
  ON dorm_allocations (school_id, dorm_id, bed_number)
  WHERE vacated_date IS NULL;
```

---

## Folder Structure (Sprint 008 additions)

```
app/(school)/[schoolSlug]/
├── boarding/
│   ├── page.tsx          ← Matron dashboard: dorm map + vacancies
│   ├── allocations/page.tsx
│   └── roster/page.tsx
├── transport/
│   ├── page.tsx          ← Transport Manager: vehicles + trips
│   ├── trips/new/page.tsx
│   └── trips/[id]/page.tsx
├── nursing/
│   ├── page.tsx          ← Nurse: visit log + inventory
│   ├── visits/new/page.tsx
│   └── inventory/page.tsx
├── kitchen/
│   ├── page.tsx          ← Cateress: meal planner + inventory
│   └── meal-plan/page.tsx
└── hr/
    ├── page.tsx           ← HR Manager: staff list + dashboard
    ├── staff/[userId]/page.tsx
    ├── appraisals/page.tsx
    └── onboarding/page.tsx

components/
├── boarding/DormMap.tsx   ← 'use client' — visual bed grid
├── transport/TripTimeline.tsx
├── nursing/VisitForm.tsx
├── kitchen/MealPlanGrid.tsx  ← 'use client' — weekly grid
└── hr/StaffDirectory.tsx
```

---

## Context Injection Update

Since Sprint 008 adds `app.current_role` to RLS policies, the Prisma client wrapper must set this:

```typescript
// server/db.ts — update prismaWithRLS
export function prismaWithRLS(schoolId: string, userId: string, role: UserRole) {
  return prisma.$extends({
    query: {
      $allModels: {
        async $allOperations({ args, query }) {
          return prisma.$transaction(async (tx) => {
            await tx.$executeRaw`
              SELECT 
                set_config('app.current_school_id', ${schoolId}, true),
                set_config('app.current_user_id', ${userId}, true),
                set_config('app.current_role', ${role}, true)
            `
            return query(args)
          })
        },
      },
    },
  })
}
```

This is a breaking change from Sprint 002 — update `server/context.ts` to pass `role` to `prismaWithRLS`.
