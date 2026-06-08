# EduManage — Product Requirements Document (PRD)

**Version:** 1.0  
**Date:** 2026-05-29  
**Author:** Architect (Session 1)  
**Status:** Approved for development

---

## 1. Product Vision

EduManage is the operating system for Kenyan private schools. It replaces every disconnected tool — WhatsApp groups, Excel fee trackers, paper registers, manual report cards — with a single, intelligent platform that connects every department and stakeholder in a school.

**Tagline:** Complete School Management. One Platform.

---

## 2. Problem Statement

Kenyan private schools (40,000+ institutions) operate with fragmented, manual workflows:

| Pain Point | Current State | EduManage Solution |
|---|---|---|
| Attendance | Paper register, re-entered into Excel | Digital register, instant reports |
| Exam results | Typed reports, manually distributed | Online entry, approved, published |
| Fee collection | Parents queue at school office or M-Pesa, manual receipt | In-app Mpesa STK Push, auto-receipt |
| Parent communication | WhatsApp groups, missed messages | Structured messaging, read receipts |
| Procurement | Paper requisitions, manual approval chains | Digital workflow, full audit trail |
| Reporting | Headteacher collects spreadsheets from each department | Consolidated real-time dashboards |
| Boarding | Manual dorm lists | Digital allocation, vacancy tracking |
| Library | Physical library only | Digital resources per subject |

---

## 3. Target Users

**Primary customer:** School Admin (Headteacher / Director) — buys the subscription  
**Primary daily users:** Teachers, Accountants, Parents  
**Secondary users:** All 15 other roles

**Market:**
- Kenya-first launch
- 40,000+ formal schools
- Initial target: private schools (CBC + IGCSE) in Nairobi, Mombasa, Kisumu
- SAM: 200 schools in Year 1 @ Ksh 420,000/school/year = Ksh 84M
- SOM (Year 1): 15 schools = Ksh 6.3M

---

## 4. Subscription Tiers

| Feature | Basic (Ksh 25K/term) | Standard (Ksh 50K/term) | Premium (Ksh 100K/term) |
|---|---|---|---|
| School onboarding + users | ✅ | ✅ | ✅ |
| Academic management | ✅ | ✅ | ✅ |
| Fee collection (Mpesa) | ✅ | ✅ | ✅ |
| Financial reports | Basic | Full | Full |
| Communications + Calendar | ❌ | ✅ | ✅ |
| Library + Resources | ❌ | ✅ | ✅ |
| Boarding + Transport + Nursing + Kitchen + HR | ❌ | ✅ | ✅ |
| AI Report Generation | ❌ | ❌ | ✅ |
| AI Anomaly Detection | ❌ | ❌ | ✅ |
| AI Smart Suggestions | ❌ | ❌ | ✅ |
| Storage (school-wide) | 50 GB | 200 GB | 1 TB |
| Priority support | ❌ | ❌ | ✅ |
| Mpesa B2C Salary Disbursement | ❌ | ❌ | ✅ |

---

## 5. User Roles & Permissions Summary

### 16 Platform Roles

| # | Role | Scope | Dept |
|---|---|---|---|
| 1 | Super Admin | Platform | EduManage |
| 2 | School Admin (Headteacher) | Whole school | Administration |
| 3 | Dean of Academics | Academic | Academic |
| 4 | Class Teacher | Assigned class | Academic |
| 5 | Subject Teacher | Assigned subjects | Academic |
| 6 | Accountant | Finance | Finance |
| 7 | Procurement Officer | Procurement | Operations |
| 8 | IT Support | Technical | Operations |
| 9 | HR Manager | HR | Administration |
| 10 | Transport Manager | Transport | Operations |
| 11 | School Librarian | Library | Academic |
| 12 | School Nurse | Nursing | Welfare |
| 13 | School Cateress | Kitchen | Welfare |
| 14 | School Matron | Boarding | Welfare |
| 15 | Parent | Own children | External |
| 16 | Student | Own records | External |

---

## 6. Core Feature Modules

### Module 1: School Administration
- School setup wizard (onboarding)
- User management (invite, deactivate, role change)
- School settings (logo, email domain, curriculum type, term dates)
- School dashboard (consolidated view of all departments)
- Approval workflows (all cross-department approvals)

### Module 2: Academic Management
- Timetable designer (Dean)
- Attendance tracking (Class Teacher)
- Exam management (Teacher → Dean → Student)
- Assignment workflow (Teacher → Student)
- Lesson plan builder (Teacher)
- Student academic progress (Student/Parent)
- Syllabus management (Dean/Teacher/Student)

### Module 3: Financial Management
- Fee structure configuration (Accountant)
- Mpesa fee collection (Parent-initiated STK Push)
- Manual payment recording (Accountant)
- Financial reports (Accountant/Headteacher)
- Payroll & PAYE (Accountant/Headteacher)
- KRA tax reports (Accountant)
- Bank statement import (Accountant)
- Budget management (all departments)
- School subscription billing (Super Admin/Headteacher)

### Module 4: Procurement
- Procurement request (any staff role)
- Multi-level approval chain (Dept Head → Procurement Officer → Accountant → Headteacher)
- Supplier order management (Procurement Officer)
- Inventory tracking (Procurement Officer + dept heads)
- Damage reporting (any dept)

### Module 5: Communications & Calendar
- Direct messaging (all roles)
- Group messaging (role-scoped)
- Unified personal calendar
- School/class/club event management
- Trip planning + approval
- Leave management (Teacher → Dean)
- Notification centre

### Module 6: Library & Resources
- School digital library (Librarian-managed)
- Teacher subject resources (per class)
- Student personal library
- Syllabus with progress tracking
- Parent library monitoring

### Module 7: AI Automation (Premium)
- Narrative report generation (Gemini)
- Anomaly detection (attendance, fees, performance)
- Timetable conflict detection
- Smart suggestions (assignments, procurement, lesson plans)

### Module 8: Operational Departments
- **Boarding:** Dorm allocation, bed management, repairs
- **Transport:** Vehicles, trips, approval, budget
- **Nursing:** Health visits, medical inventory, alerts
- **Kitchen:** Meal planning, inventory, budget
- **HR:** Staff records, documents, appraisals, onboarding

---

## 7. Non-Functional Requirements

| Requirement | Target |
|---|---|
| Page load (initial) | < 2 seconds on 10Mbps |
| API response (p95) | < 300ms |
| Uptime | 99.5% (Vercel SLA) |
| Data isolation | Zero cross-tenant data leakage |
| Mobile responsiveness | Full mobile support (most users on phone) |
| File upload max | 50MB per file |
| Concurrent users (MVP) | 500 per school, 5,000 platform-wide |
| Languages | English (v1); Swahili (v2) |

---

## 8. Security Requirements

- All staff/student logins: school-assigned email only
- All API endpoints: JWT auth enforced
- Database: Row-Level Security on all tenant tables
- File storage: No public S3 buckets, presigned URLs with 15-min expiry
- Exam papers: Encrypted at rest, time-locked access
- PII: Not logged; audit logs store entity IDs only
- Webhooks: Signature verification (Mpesa + Stripe + Clerk)
- HTTPS: Enforced by Vercel

---

## 9. Compliance

- **Kenya Data Protection Act 2019:** Student data treated as sensitive. No sharing with third parties.
- **KRA Tax Compliance:** PAYE, VAT, WHT reports generated for manual iTax filing.
- **Mpesa Daraja:** Safaricom compliance requirements for STK Push + C2B.
- **Stripe:** PCI-DSS handled by Stripe; EduManage never stores card data.

---

## 10. Delivery Roadmap

| Sprint | Deliverable | Duration (est.) | Revenue Gate |
|---|---|---|---|
| 001 | Architecture & Planning | 1 week | No |
| 002 | Auth, Multi-Tenancy, Onboarding | 2 weeks | No |
| 003 | Academic Core | 3 weeks | No |
| 004 | Financial & Mpesa | 3 weeks | **YES — first billing** |
| 005 | Communications & Calendar | 2 weeks | YES |
| 006 | Library & Resources | 2 weeks | YES |
| 007 | AI Automation | 2 weeks | YES (Premium) |
| 008 | Boarding/Transport/Nursing/Kitchen/HR | 3 weeks | YES (Full platform) |

**Total estimated build time:** 18 weeks (with 2 developers working in parallel on UI and backend from Sprint 003 onward)

**First revenue:** After Sprint 004 — target 15 schools @ Ksh 50,000/term = **Ksh 750,000/term**

---

## 11. Success Metrics (Year 1)

| Metric | Target |
|---|---|
| Schools onboarded | 15 (by Sprint 004 launch) |
| Schools at end of Year 1 | 50 |
| Monthly Active Users | 5,000+ |
| Net Revenue Retention | >110% (upsells to Premium) |
| Mpesa payment volume processed | Ksh 50M+ |
| NPS (from Headteachers) | >50 |
| Churn rate | <5% per term |

---

## 12. Out of Scope (v1)

- Biometric attendance hardware integration
- Direct KRA iTax API filing
- SMS notifications (Africa's Talking — v2)
- Swahili language support
- Mobile app (React Native — v2; PWA in v1)
- Multi-country / multi-currency beyond KES + USD
- Video conferencing integration
- Parent-teacher video calls
- AI exam paper generation
