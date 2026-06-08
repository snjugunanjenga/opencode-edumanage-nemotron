# QUESTIONS.md — EduManage

All questions from Session 1 have been answered by the Operator. Closed.

---

## Q-001: KRA iTax Direct API Integration
**Status:** ANSWERED  
**Answer:** Option A — Generate KRA-formatted reports (PAYE, VAT, WHT) as PDF/CSV for manual iTax upload. No direct API integration in v1.  
**Decision recorded:** DEC-009

## Q-002: Per-User File Storage Quotas
**Status:** ANSWERED  
**Answer:** School-wide quota per subscription tier: Basic 50GB / Standard 200GB / Premium 1TB. No per-user cap enforced separately.  
**Decision recorded:** DEC-010

## Q-003: Exam Paper Security Model
**Status:** ANSWERED  
**Answer:** Option A — Platform stores encrypted exam file. Dean sets release timestamp. File accessible only after timestamp passes and only to authorised roles.  
**Decision recorded:** DEC-011

## Q-004: Biometric Attendance Hardware Integration
**Status:** ANSWERED  
**Answer:** Option A — Manual digital attendance via app in v1. Hardware integration deferred to v2.  
**Decision recorded:** DEC-012

## Q-005: AWS S3 Region
**Status:** ANSWERED  
**Answer:** eu-west-1 (Ireland) as primary. GCP Cloud Storage (europe-west1) evaluated as a secondary option/fallback. Decision: use S3 eu-west-1 for v1; add GCP bucket replication in v2 if data-residency pressure increases.  
**Decision recorded:** DEC-013

## Q-006: Salary Payment Method
**Status:** ANSWERED  
**Answer:** Option B — EduManage generates payslips + PAYE calculations; school disburses salary manually via their own bank. Platform does NOT initiate B2C Mpesa salary payments.  
**Decision recorded:** DEC-014

---

## New Open Questions (Session 2)

## Q-007: Quiz Auto-Grading — Question Types
**Raised:** 2026-05-29  
**Raised by:** Architect  
**Impact if unresolved:** Cannot finalise quiz engine schema or grading logic.  
**Options:**
- A: Multiple choice + True/False only (auto-gradeable)
- B: Multiple choice + True/False + Short answer (short answer manually graded by teacher)
- C: All of the above + Essay (essay AI-assisted grading via Gemini — Premium only)  
**Status:** Open  
**Answer:** Pending. Architect recommends Option B for v1, Option C added in Sprint 007 AI layer.

## Q-008: Analytics Data Retention Period
**Raised:** 2026-05-29  
**Raised by:** Architect  
**Impact if unresolved:** Cannot set DB partitioning or archiving strategy for analytics tables.  
**Options:**
- A: Retain all data indefinitely (simple, storage cost grows)
- B: Retain raw events 2 years, roll up to monthly summaries after  
**Status:** Open  
**Answer:** Pending. Architect recommends Option B.
