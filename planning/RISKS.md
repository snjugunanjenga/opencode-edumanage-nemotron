# RISKS.md — EduManage

---

## RISK-001: Mpesa Daraja API Reliability

**Severity:** High  
**Probability:** Medium  
**Description:** Safaricom's Daraja API experiences periodic downtime, rate limiting, and callback delivery failures. School fee collection is mission-critical.  
**Impact:** Parents cannot pay fees. School accountants cannot reconcile. Platform loses credibility with first schools.  
**Mitigation:** Idempotent callback handlers. Retry queue via Inngest. Manual payment entry fallback for accountants. Webhook signature verification. Sandbox-to-production testing before each school onboarding.  
**Status:** Open

---

## RISK-002: Clerk Pricing at Scale

**Severity:** Medium  
**Probability:** Medium  
**Description:** Clerk charges per Monthly Active User. At 500 schools × 500 users avg = 250,000 MAU, Clerk costs become significant.  
**Impact:** Margin erosion at scale. Could force auth migration.  
**Mitigation:** Monitor MAU vs Clerk invoice monthly from 50 schools. Evaluate self-hosted Clerk or migration to Auth.js at 5,000 MAU threshold. Document migration path now.  
**Status:** Open

---

## RISK-003: KRA Tax Compliance Complexity

**Severity:** High  
**Probability:** High  
**Description:** Kenyan tax law (PAYE, VAT, WHT) is complex and changes periodically. Schools are liable. If EduManage generates incorrect tax calculations, schools face KRA penalties.  
**Impact:** Legal liability for schools. Reputational damage to EduManage. Potential regulatory scrutiny.  
**Mitigation:** Tax calculations reviewed by a qualified Kenyan accountant before Sprint 004 ships. Tax reports marked "for review — not a substitute for professional tax advice." KRA iTax direct integration deferred to v2. Platform generates reports; school accountant files manually. Track KRA regulation changes via KRA developer portal.  
**Status:** Open

---

## RISK-004: Scope Creep — 15+ Role Wishlist

**Severity:** High  
**Probability:** High  
**Description:** The platform roadmap contains 22+ improvement items across 15+ roles. Building all simultaneously with a team of 4 risks delivering nothing well.  
**Impact:** Delayed launch. Technical debt. Unmaintainable codebase. No revenue.  
**Mitigation:** Sprint structure enforces delivery in phases. Sprints 002–004 are the chargeable MVP. Sprints 005–008 are premium features. No Sprint N+1 starts until Sprint N acceptance criteria pass. All new feature requests go to QUESTIONS.md, not directly to Builder.  
**Status:** Mitigated (by sprint structure)

---

## RISK-005: S3 Data Residency — No AWS Kenya Region

**Severity:** Medium  
**Probability:** Low  
**Description:** AWS has no Kenya region. Student data (minors) stored in Ireland (eu-west-1) or South Africa (af-south-1). Kenya's Data Protection Act 2019 requires reasonable data protection but does not explicitly mandate local residency.  
**Impact:** Potential future compliance issue if Kenya tightens data residency rules. Parent/school trust concern.  
**Mitigation:** Use af-south-1 (Cape Town) as primary — closest African region. Document data location in privacy policy. Monitor Kenya DPA regulatory developments. Architect S3 bucket structure to enable future region migration without application changes.  
**Status:** Open

---

## RISK-006: Exam Paper Security

**Severity:** High  
**Probability:** Medium  
**Description:** Teachers upload exam papers (.docx) to the platform before the exam date. A breach or misconfigured permission could expose papers to students.  
**Impact:** Exam integrity compromised. School loses trust. EduManage liable.  
**Mitigation:** Exam papers stored in a separate S3 bucket prefix with no public access. Presigned URLs generated only after Dean approval + only for authorised roles (Dean, Headteacher). Files encrypted at rest (S3 SSE-S3). Password required on every login when accessing exam section. Time-locked release — papers only accessible to students after Dean sets release timestamp.  
**Status:** Open

---

## RISK-007: Team Bus Factor

**Severity:** High  
**Probability:** Medium  
**Description:** Team of 4 with Technical Lead as single point of knowledge for the codebase.  
**Impact:** If Technical Lead is unavailable, platform development halts.  
**Mitigation:** All architecture documented in this planning pack. Builder agent (Claude Code) can implement from handoff-prompt.md without oral knowledge transfer. README and CONTEXT.md kept current. Code review shared between Technical Lead and DevOps.  
**Status:** Open

---

## RISK-008: Multi-Currency / Forex Exposure

**Severity:** Low  
**Probability:** Low  
**Description:** Stripe subscription payments from international schools are in USD/GBP. EduManage revenue reported in KES. Forex volatility affects forecasting.  
**Impact:** Revenue unpredictability. Accounting complexity.  
**Mitigation:** Stripe Payouts in USD to a USD account. Convert to KES on a schedule. Deferred concern until first international school signs.  
**Status:** Accepted (low priority)
