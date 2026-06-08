# DOMAIN.md — EduManage

**Updated:** 2026-05-29

---

## Business Context

EduManage is a B2B SaaS product sold to Kenyan private schools on a per-term subscription. The founding team is 4 people (Technical Lead, DevOps, Security Engineer, Product Manager/Business Analyst). Revenue model: Basic Ksh 25,000/term, Standard Ksh 50,000/term, Premium Ksh 100,000/term. Launch target: 15 schools at Standard tier. Funding: bootstrapping + angel investors.

## The Problem

Kenyan private schools run on disconnected tools: physical registers, WhatsApp groups, Excel fee trackers, paper procurement forms, and manual report cards. There is no unified system that connects academic performance, fee payment status, staff management, procurement, boarding, and parent communication. EduManage is the single platform that replaces all of it.

## Curriculum Systems Supported

- **CBC (Competency Based Curriculum):** Kenya's current national curriculum. Grades: PP1, PP2, Grade 1–9 (Primary), Grade 10–12 (Senior Secondary). Assessment: formative (project-based, portfolio) + summative.
- **IGCSE / British System:** Used in international and elite private schools. Years: Year 1–13. Includes O-Level (IGCSE) and A-Level streams.

## Users and Roles (all 16)

### Platform Level
| Role | Scope | Key Responsibilities |
|---|---|---|
| Super Admin | Platform-wide | Onboard schools, manage subscriptions, platform health |

### School Level — Administration
| Role | Scope | Key Responsibilities |
|---|---|---|
| School Admin (Headteacher/Principal) | Whole school | All operations, approve salaries/budgets/payments, receive dept reports |
| IT Support | Whole school | System maintenance, user account management within school |
| HR Manager | Whole school | Staff records, contracts, leave approvals (alongside Dean) |

### School Level — Academic
| Role | Scope | Key Responsibilities |
|---|---|---|
| Dean of Academics | Academic ops | Timetable design, teacher-class assignments, leave + replacement scheduling, exam approval |
| Class Teacher | Assigned class | Attendance, class performance, class calendar, parent communication |
| Subject Teacher | Assigned subjects | Lesson plans, assignments, exam setting, marking |

### School Level — Financial
| Role | Scope | Key Responsibilities |
|---|---|---|
| Accountant | Finance dept | Fee collection, financial reports, budget approvals, salary payments, KRA tax compliance, Mpesa + bank statement downloads |

### School Level — Operations
| Role | Scope | Key Responsibilities |
|---|---|---|
| Procurement Officer | Procurement dept | Purchase requests, supplier orders, inventory, delivery tracking, damages |
| Transport Manager | Transport dept | Trip planning, vehicle management, budget + reports to accountant |
| School Librarian | Library | Resource uploads, student library access monitoring, syllabus management |
| School Nurse | Nursing dept | Student health visits, medical records, procurement of medical supplies |
| School Cateress | Kitchen dept | Meal planning, kitchen inventory, procurement requests |
| School Matron | Boarding dept | Dorm allocation, bed management, boarding student welfare, repairs |

### Parent / Student Level
| Role | Scope | Key Responsibilities |
|---|---|---|
| Parent | Own children only | View academic reports, pay fees, download Mpesa statements, monitor library activity |
| Student | Own records only | View scores, submit assignments, access library, join clubs, view timetable |

---

## Business Rules

### Tenancy
- Every record belongs to exactly one school (school_id). No cross-tenant data access ever.
- A user belongs to exactly one school (except Super Admin who is platform-scoped).

### Authentication
- All staff must use school-assigned email addresses for login.
- Students must use school-assigned email addresses.
- Parents use personal email; linked to student records by Headteacher/Admin.
- Every user receives a platform-generated unique_id at registration.

### Academic
- A school operates on a term calendar (3 terms/year standard Kenya). 
- A Class has exactly one Class Teacher. A subject can have multiple teachers across classes.
- Attendance is recorded per student per day per class.
- Exam results require Dean of Academics approval before being visible to students/parents.
- Exam papers (.docx) submitted by teachers require password + Dean approval workflow. Papers are encrypted at rest.
- A teacher requesting leave requires Dean approval + replacement teacher assignment before leave is granted.
- Timetable changes by Dean propagate to all affected users' calendars.

### Financial
- Fee payments are term-based. Each student has a fee balance per term.
- Mpesa STK Push is used for parent fee payments. C2B for school bulk payments.
- All financial transactions generate an immutable audit record.
- Salary payments require Headteacher approval.
- Budget requests from departments require Accountant review + Headteacher approval above a threshold.
- KRA compliance: PAYE on staff salaries, VAT on applicable services, Withholding Tax on supplier payments. Reports generated in-platform; filing assumed manual via iTax (direct API integration deferred to v2).
- School subscription payments: Mpesa (Kenyan schools) or Stripe (international).

### Procurement
- Any department (teacher, nurse, cateress, matron, transport) can raise a procurement request.
- Requests route: Requestor → Department Head → Procurement Officer → Accountant (budget check) → Headteacher (above threshold).
- Orders placed only after full approval chain complete.
- Inventory updated automatically on delivery confirmation.
- Damaged items reported separately; triggers reorder workflow.

### Communications
- Direct messages: 1-to-1 between any two users in the same school.
- Group messages: role-scoped groups (e.g., "All Parents", "Grade 5 Parents", "Staff").
- Before sending to a group of >10 people, a confirmation dialog is shown ("You are about to message N people. Confirm?").
- Message history is stored and searchable.
- Notifications sent via in-app + email (Resend) for: new messages, exam results released, fee reminders, assignment due dates, approval requests.

### Library
- Resources tagged by subject, class, curriculum type.
- Students have a max storage quota (to be defined — open question Q-002).
- School librarian monitors student upload/download activity.
- Teacher-uploaded resources are visible to their assigned classes only.
- Syllabus documents visible to all teachers and students in relevant class.

### Boarding (Matron)
- Boarding students are flagged at registration.
- Matron allocates dorm + bed. One student per bed.
- Dorm repairs raised as procurement requests to Procurement Officer.
- Bed purchases raised as procurement requests.

### Clubs & Groups
- Students can join/leave clubs. Teachers can be patrons.
- Club events appear on member calendars.
- Trip planning requires: Teacher initiates → Transport Manager → relevant HODs → Headteacher approval.

### AI Automation (Sprint 007)
- Gemini API used for: narrative report generation, fee defaulter analysis, attendance anomaly alerts, timetable conflict detection, procurement spend summaries.
- AI suggestions are advisory only — no autonomous financial or academic actions.
- AI features are Premium tier only.

---

## Terminology

| Term | Definition |
|---|---|
| Tenant | One school instance on the platform |
| Term | Academic term (approx 3 months). Kenya has 3 terms/year. |
| Stream | Parallel class within a grade (e.g., Grade 5A, Grade 5B) |
| CBC | Competency Based Curriculum — Kenya's national system |
| IGCSE | International General Certificate of Secondary Education — British system |
| STK Push | Safaricom Mpesa "Send to Till" payment prompt to customer's phone |
| C2B | Mpesa Customer-to-Business payment |
| Daraja | Safaricom's developer API platform for Mpesa integration |
| PAYE | Pay As You Earn — Kenyan income tax on salaries |
| VAT | Value Added Tax — 16% in Kenya |
| WHT | Withholding Tax — deducted on supplier payments |
| KRA | Kenya Revenue Authority — tax body |
| iTax | KRA's online tax filing portal |
| unique_id | Platform-assigned identifier per user (e.g., EDU-SCH001-STU-00234) |
| HOD | Head of Department |
| Dean | Dean of Academics |

## Constraints

- Kenya-first launch: KES currency, Mpesa as primary payment rail, KRA tax rules
- No AWS Kenya region: S3 data stored in eu-west-1 (Ireland) or af-south-1 (Cape Town) — operator to confirm
- Team of 4: architecture must be simple enough for 2 developers to maintain
- Budget: bootstrapped — no expensive managed services in Sprint 1-3
- Compliance: student data is sensitive (minors); GDPR-equivalent care required even without formal Kenyan data protection enforcement
