# Requirements — Sprint 008: Boarding, Transport, Nursing, Kitchen & HR

**Project:** EduManage  
**Sprint:** 008  
**Status:** Approved  
**Last updated:** 2026-05-29  
**Depends on:** Sprint 007 complete  
**Significance:** Full platform complete — all 16 roles fully operational

---

## Sprint Goal

Complete the five remaining operational departments: School Matron (boarding), Transport Manager, School Nurse, School Cateress (kitchen), and HR Manager. Each integrates with the existing procurement system (Sprint 005), financial management (Sprint 004), calendar (Sprint 003), and communications (Sprint 005). EduManage is the complete product after this sprint.

---

## FR-001: Boarding (School Matron)

**Role:** MATRON

- **Dorm management:** Create dorms — name, building, total beds, gender (MALE/FEMALE/MIXED), active status
- **Bed allocation:** Matron assigns boarding student to dorm + bed number; one student per active bed enforced at DB level (partial unique index on `(schoolId, dormId, bedNumber)` WHERE `vacatedDate IS NULL`)
- **Vacancy tracking:** Real-time view of occupied/available beds per dorm
- **Student transfer:** Move student to different dorm/bed (vacate old, allocate new)
- **Dorm repairs:** Matron raises procurement request (category: REPAIR) → routes through standard chain
- **Bed purchases:** Matron raises procurement request (category: EQUIPMENT)
- **Cleaning roster:** Matron records cleaning staff assignments (name + shift + weekdays) — text-based, no login
- **Boarding flag:** `Student.isBoarder` set at registration; Matron can update

---

## FR-002: Transport (Transport Manager)

**Role:** TRANSPORT_MANAGER

- **Vehicle registry:** Name, plate number, capacity, driver name, insurance expiry date
- **Trip planning:** Any teacher can initiate a trip; Transport Manager assigns vehicle + confirms logistics
- **Trip approval chain:** Transport Manager → relevant HOD → Headteacher
- **Trip manifest:** Class Teacher confirms student list + guardian consent per student
- **Trip calendar event:** Auto-created (eventType: TRIP) for all students on manifest + chaperones on approval
- **Insurance alerts:** Inngest weekly check — vehicles with insurance expiring ≤30 days → notify Transport Manager + Headteacher
- **Term budget:** Transport Manager submits → Accountant → Headteacher approval
- **Trip reports:** Completed trips list, actual vs estimated cost, downloadable PDF

---

## FR-003: Nursing (School Nurse) — Confidential

**Role:** SCHOOL_NURSE  
**Access:** Only SCHOOL_NURSE + HEADTEACHER can view health records — enforced at BOTH tRPC AND RLS level

- **Health visit log:** Date, student, complaint, treatment given, outcome, follow-up required flag + date
- **Health alerts:** Nurse flags follow-up required → in-app notification to Parent + Headteacher + Class Teacher (notification only — not the health record content)
- **Medical inventory:** Name, quantity, unit, reorder level, expiry date
- **Procurement:** Nurse raises medical supply request (category: MEDICAL) → standard chain
- **Immunisation records:** Optional log of student immunisation history (date, vaccine, next due)
- **Confidentiality:** PARENT receives alert notifications but CANNOT view visit record content

---

## FR-004: Kitchen (School Cateress)

**Role:** CATERESS

- **Meal planner:** Weekly plan by day + meal type (BREAKFAST/LUNCH/DINNER/SNACK); published plan visible to all school users on their calendar
- **Kitchen inventory:** Ingredients tracked by quantity, unit, reorder level
- **Auto-procurement draft:** When item falls below reorder level → Inngest creates DRAFT procurement request for Cateress to review and manually submit
- **Term budget:** Cateress submits → Accountant → Headteacher approval
- **Monthly report:** Meals served + estimated cost → PDF submitted to Accountant

---

## FR-005: HR Management (HR Manager)

**Role:** HR_MANAGER

- **Staff records:** Employment start date, contract type (PERMANENT/CONTRACT/PART_TIME), department, salary grade, probation end date, contract end date, emergency contact
- **Staff documents:** Upload: contracts, teaching certificates, ID copies → S3; HR_MANAGER + HEADTEACHER view only
- **Contract expiry alerts:** Inngest monthly check — contracts expiring ≤60 days → notify HR Manager
- **Leave overview:** Consolidated view of all school leaves (Sprint 005 data)
- **Performance appraisals:** Annual cycle — HR creates → staff self-appraises → line manager reviews → Headteacher approves
- **Onboarding checklist:** New hire → HR marks off: contract signed, ID verified, school email set up, added to payroll, induction completed
- **Payroll handoff:** HR confirms active headcount for month → Accountant notified to run salary

---

## Procurement Integration (All 5 Departments)

All departments use the existing `procurement.*` tRPC router from Sprint 005. Sprint 008 adds:
- New `department` values: BOARDING, TRANSPORT, NURSING, KITCHEN, HR
- New `category` values: REPAIR, EQUIPMENT, MEDICAL, FOOD, UNIFORM
- Department HOD routing: MATRON → HT (direct), NURSE → HT (direct), CATERESS → HT (direct), TRANSPORT_MANAGER → HT, HR_MANAGER → HT

---

## Non-Functional Requirements

- Health visit confidentiality: tested with explicit cross-role access denial
- Dorm double-allocation: prevented at database level (partial unique index) — not application level
- Trip manifest: guardian consent required before student included in final confirmed manifest
- Auto-procurement drafts (kitchen): DRAFT status only — Cateress must manually review and submit
- HR documents: never included in any API response outside `hr.getDocuments` — no accidental leakage
