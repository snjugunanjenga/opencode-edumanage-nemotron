# Requirements — Sprint 007: AI Automation Layer

**Project:** EduManage  
**Sprint:** 007  
**Status:** Approved  
**Last updated:** 2026-05-29  
**Depends on:** Sprint 006 complete  
**Tier gate:** Premium only (DEC-005)

---

## Sprint Goal

Premium-tier schools get AI-powered automation via Google Gemini 1.5 Pro — narrative report generation, anomaly detection, smart feedback suggestions, timetable analysis, lesson plan assistance, and financial narrative — all running asynchronously through Inngest. No AI output is ever published without human approval.

---

## FR-001: Narrative Term Report Generation

**Roles:** Dean or Headteacher triggers; Dean + Class Teacher review; Headteacher approves; Parent receives

- Dean clicks "Generate Term Reports" for a class/year group
- Per-student Inngest job: fetches exam averages, quiz scores, attendance %, assignment completion rate, teacher comments
- Gemini generates 200–400 word narrative per student (qualitative language only — no raw scores)
- Report stored as `AIReport` with `status = GENERATED`
- Dean + Class Teacher review and edit if needed
- Headteacher approves → `status = APPROVED` → parent notified + can view in-app

---

## FR-002: Anomaly Detection (Nightly — Premium Schools Only)

Four automatic detectors run nightly at 21:00 KE:

| Detector | Trigger Condition | Notified |
|---|---|---|
| Attendance anomaly | >20% absence rate in current term | Class Teacher + Dean |
| Fee defaulter | >50% outstanding balance at Week 6 of term | Accountant + Headteacher |
| Score drop | Score drops >20 percentage points vs last term | Class Teacher + Dean |
| Procurement overspend | Department at >110% of approved budget | Accountant |

Each creates `AIInsight` record + in-app notification. Idempotent — one insight per entity per term per type.

---

## FR-003: Smart Assignment Feedback Suggestions

**Roles:** Teacher (on assignment marking page)

- After entering a score, "Suggest Feedback" button appears
- On click: async Inngest job → Gemini returns 3 feedback options (encouraging / direct / detailed)
- Teacher selects, edits, saves
- Teacher can ignore and write own feedback — never blocking
- AI suggestions expire after 24 hours

---

## FR-004: Timetable AI Review

**Roles:** Dean

- Dean clicks "AI Review Timetable" on the timetable designer
- Inngest job: sends full timetable data to Gemini
- Gemini analyses for: teacher workload imbalance, same-subject back-to-back periods, missing required subjects per CBC/IGCSE minimum, students with no breaks
- Returns structured JSON with issues + suggested fixes
- Dean sees sidebar panel with flagged issues; can dismiss each

Note: Hard conflict detection (double-booking) remains deterministic and synchronous (Sprint 003). This is a soft/advisory review only.

---

## FR-005: Lesson Plan Assistant

**Roles:** Class Teacher + Subject Teacher

- Teacher enters: subject, class, topic, duration (minutes), 2–3 learning objectives
- Clicks "Generate Draft" — async Inngest job → Gemini generates full lesson plan
- Returns structured plan: intro, main activity (with steps), assessment activity, resources needed, homework suggestion
- Teacher reviews, edits any section, saves as LessonPlan (DRAFT)
- Download as .docx uses existing lesson plan export from Sprint 003

---

## FR-006: Financial AI Narrative

**Roles:** Accountant triggers; Headteacher views

- After any financial report is generated (Sprint 004), Accountant clicks "Generate AI Summary"
- Inngest job → Gemini → 150–300 word executive summary of the financial data
- Summary appended to the existing PDF report
- Shown on Headteacher's financial dashboard overview

---

## FR-007: Usage Controls & Audit

- All AI features gated behind `premiumProcedure` middleware
- Every Gemini call logged: tokens used, latency, success/failure
- All AI-generated content: persistent banner "AI-generated — please review before publishing"
- No AI content auto-published — human approval always required
- AI usage dashboard (Headteacher + IT Support): requests made, tokens consumed, reports generated

---

## Premium Tier Enforcement

If a non-Premium school hits any AI endpoint:
```
HTTP 403: "This feature requires a Premium subscription. 
           Upgrade at: /settings/subscription"
```

---

## Non-Functional Requirements

- Term report generation: all students in a class completed within 5 minutes (Inngest concurrency 5)
- Feedback suggestions: returned within 10 seconds
- Anomaly detection: all schools processed within 30 minutes of 21:00 KE trigger
- All Gemini calls: timeout at 30 seconds, retry once on timeout
- Token budget: log alert if single school exceeds 1M tokens/month
