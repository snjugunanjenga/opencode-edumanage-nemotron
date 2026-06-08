# Acceptance Criteria — Sprint 004: Financial Management & Mpesa

**Last updated:** 2026-05-29

---

## AC-001: Fee Structure

**Given** Accountant creates a fee structure for Grade 7, Term 2 with 3 items (Tuition 20,000 / Activity 2,000 / Meals 5,000)  
**When** Headteacher approves it  
**Then** `FeeStructure.isActive = true`; all Grade 7 students have `StudentFeeBalance.totalOwed = 27,000` for Term 2

---

## AC-002: Mpesa STK Push

**Given** Parent taps "Pay Now" for Ksh 5,000, enters phone 2547XXXXXXXX  
**When** they confirm  
**Then** within 3 seconds they see a "Check your phone for M-PESA prompt" screen; `MpesaStkRequest` row created with status PENDING

**Given** Safaricom sends a SUCCESS callback with receipt M123456789  
**When** the Inngest job processes it  
**Then** `FeePayment` exists with `mpesaReceiptNo = M123456789, status = COMPLETED`; `StudentFeeBalance.totalPaid` increased by 5,000; parent receives email receipt

**Given** the SAME callback is received again (Safaricom retry)  
**When** Inngest processes it  
**Then** NO duplicate `FeePayment` is created — idempotency confirmed

**Given** callback with `ResultCode != 0` (payment failed/cancelled)  
**When** processed  
**Then** `MpesaStkRequest.status = FAILED`; parent sees "Payment cancelled" notification

---

## AC-003: Manual Payment

**Given** Accountant records a cash payment of Ksh 3,000 for student Achieng  
**When** saved  
**Then** `FeePayment` created with `paymentMethod = CASH`; `StudentFeeBalance` updated; AuditLog entry created

---

## AC-004: Statement Download

**Given** Accountant filters payments for Class 7A, Term 2, Jan–June 2026  
**When** they click "Download PDF"  
**Then** a PDF is returned with school header, filtered payment list, total collected, balance outstanding

**Given** Parent views their child's fee statement  
**When** they click "Download Mpesa Statement"  
**Then** PDF contains ONLY their child's payments — no other student data visible

---

## AC-005: Payroll & PAYE

**Given** Teacher Kamau has gross salary Ksh 45,000  
**When** PAYE is calculated  
**Then** result matches: PAYE = Ksh 7,133, NHIF = Ksh 1,000, NSSF = Ksh 2,160, Net = Ksh 34,707 (±Ksh 5 rounding)

**Given** Accountant creates a salary run for May 2026, enters all staff salaries  
**When** Headteacher approves  
**Then** `SalaryRun.status = APPROVED`; Inngest job triggered; each staff member receives payslip PDF by email

---

## AC-006: KRA Tax Reports

**Given** Accountant generates PAYE report for April 2026  
**When** downloaded  
**Then** CSV file has correct KRA P10 column headers and one row per staff member with correct figures; file header reads "For professional review — not for direct iTax submission"

---

## AC-007: Bank Reconciliation

**Given** Accountant uploads a bank CSV with 50 transactions, 30 of which match existing FeePayments by amount + date ±1 day  
**When** Inngest `bankReconcile` job runs  
**Then** 30 `BankTransaction` rows have `status = MATCHED`; 20 have `status = UNMATCHED`; Accountant sees unmatched list for manual review

---

## AC-008: Subscription Billing

**Given** a school on TRIAL with `trialEndsAt` in the past  
**When** the daily `subscriptionExpiry` Inngest job runs  
**Then** `SchoolSubscription.status = PAST_DUE`; Headteacher receives "Trial expired" in-app notification

**Given** Headteacher completes Stripe Checkout for Standard tier  
**When** Stripe sends `checkout.session.completed` webhook  
**Then** `SchoolSubscription.tier = STANDARD, status = ACTIVE, currentPeriodEnd` set correctly

---

## AC-009: Financial Immutability

**Given** a `FeePayment` with `status = COMPLETED`  
**When** any tRPC mutation attempts to UPDATE its `amount` or `status`  
**Then** tRPC returns `FORBIDDEN: "Completed payments are immutable"`

---

## Definition of Done

- All AC-001 through AC-009 pass
- PAYE calculation unit tests pass with KRA published examples (3+ test cases)
- Mpesa idempotency integration test passes
- Stripe webhook integration test passes (using Stripe CLI)
- No financial data crosses tenant boundary (RLS verified)
- All new tables have RLS policies
- PDF receipts generated correctly for Mpesa + manual payments
- `STATE.md` updated, `CONTEXT.md` Sprint History updated
