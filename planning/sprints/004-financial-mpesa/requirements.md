# Requirements — Sprint 004: Financial Management & Mpesa

**Project:** EduManage  
**Sprint:** 004  
**Status:** Approved — ready for Builder  
**Last updated:** 2026-05-29  
**Depends on:** Sprint 003 complete  
**Revenue gate:** ✅ First billable sprint — schools pay Ksh 50,000/term after this ships

---

## Sprint Goal

Enable schools to collect fees via Mpesa STK Push, manage the full financial lifecycle (fee structures, balances, reconciliation), generate payslips and PAYE reports, produce KRA-compliant tax exports, and allow schools to subscribe to EduManage via Mpesa or Stripe. After this sprint, EduManage can charge its first 15 schools.

---

## FR-001: Fee Structure Configuration

**Priority:** P0 | **Roles:** Accountant configures; Headteacher approves

- Accountant creates fee items per class per term: tuition, activity fee, transport, meals, boarding, library
- Each fee item has: name, amount (KES), mandatory/optional flag, due date
- Fee structure can be cloned from previous term
- Headteacher approves fee structure before it goes live
- Students see their applicable fee items on login

**Data:**
```
FeeStructure { id, schoolId, termId, classId, approvedByUserId, isActive, createdAt }
FeeItem { id, feeStructureId, schoolId, name, amount, isMandatory, dueDate, category }
StudentFeeBalance { id, schoolId, studentId, termId, totalOwed, totalPaid, balance, updatedAt }
```

---

## FR-002: Mpesa STK Push (Parent Fee Payment)

**Priority:** P0 | **Roles:** Parent initiates; Accountant views

**Flow:**
1. Parent taps "Pay Now" on fee statement → enters Mpesa phone number → confirms amount
2. App calls `financial.initiatePayment` tRPC → server calls Daraja STK Push API
3. Safaricom sends payment prompt to parent's phone
4. Parent enters M-PESA PIN
5. Safaricom sends C2B callback to `/api/webhooks/mpesa`
6. Inngest job `processMpesaPayment` runs (idempotent — checks `mpesaReceiptNo`)
7. `FeePayment` created, `StudentFeeBalance` updated, receipt PDF emailed via Resend
8. Parent sees updated balance; Accountant sees new payment in dashboard

**Idempotency rule:** If `mpesaReceiptNo` already exists in DB → skip, return 200 OK (Safaricom retries callbacks)

**Data:**
```
FeePayment {
  id, schoolId, studentId, termId, feeItemId?,
  amount, currency (KES), paymentMethod (MPESA | BANK | CASH | CHEQUE),
  mpesaReceiptNo?, mpesaPhoneNo?, mpesaTransactionDate?,
  stripePaymentId?, reference?, status (PENDING|COMPLETED|FAILED|REVERSED),
  recordedByUserId?, paidAt, createdAt
}
MpesaStkRequest { id, schoolId, checkoutRequestId, merchantRequestId, studentId, amount, phoneNo, status, createdAt }
```

**Business rules:**
- STK Push timeout: 30 seconds. If no callback in 5min, mark as FAILED
- Minimum payment: Ksh 10 (Safaricom minimum)
- Parent can make partial payment — balance updated accordingly
- All Mpesa callbacks verified with Daraja credentials before processing

---

## FR-003: Manual Payment Entry

**Priority:** P0 | **Roles:** Accountant only

- Accountant records: cash, cheque, bank transfer payments
- Fields: student, amount, payment method, reference number, date, receipt number
- Manual payments generate same PDF receipt as Mpesa
- All manual payments logged in AuditLog

---

## FR-004: Mpesa Statement Download

**Priority:** P0 | **Roles:** Accountant (all students); Parent (own child)

- Filter by: date range, student, payment type, fee category
- Export as: PDF (formatted statement) or CSV (raw data)
- PDF uses school letterhead (school name + logo from settings)
- Accountant can download school-wide statement or per-student
- Parent can download only their child's statement

---

## FR-005: Fee Reminders

**Priority:** P1 | **Roles:** Inngest scheduled job

- Runs every Monday + 7 days before term end date
- Finds all students where `StudentFeeBalance.balance > 0`
- Sends in-app notification + email (Resend) to parent: "Outstanding balance: Ksh X"
- School can configure: enable/disable, custom message
- Accountant sees "Sent X reminders" in dashboard

---

## FR-006: Financial Reports

**Priority:** P0 | **Roles:** Accountant generates; Headteacher views

| Report | Contents | Format |
|---|---|---|
| Term Income Report | Total collected, outstanding, breakdown by class and fee category | PDF + Excel |
| Expenditure Report | Procurement costs + salary totals per term | PDF + Excel |
| Cash Flow Statement | Monthly inflows vs outflows | PDF + Excel |
| Budget vs Actual | Per department, per term | PDF + Excel |
| Fee Defaulters List | Students with outstanding balance > 0 at term close | PDF + Excel |
| Mpesa Reconciliation | Matched/unmatched Mpesa transactions | PDF + CSV |

All reports: date-range filterable, school-branded PDF header

**Data:**
```
FinancialReport { id, schoolId, reportType, parameters Json, generatedByUserId, s3Key, createdAt }
```

---

## FR-007: Payroll & PAYE

**Priority:** P0 | **Roles:** Accountant prepares; Headteacher approves

**Salary run flow:**
1. Accountant creates salary run for a month
2. System pulls all active staff (from HR module or manual entry in this sprint)
3. Accountant enters/confirms: basic pay, allowances, deductions per staff member
4. System auto-calculates PAYE using Kenya 2024 tax bands (see below)
5. System calculates NHIF, NSSF deductions
6. Net pay computed: `basic + allowances - PAYE - NHIF - NSSF - other_deductions`
7. Headteacher reviews and approves salary run
8. Payslip PDFs generated + emailed to each staff member via Resend
9. School disburses salaries manually via their bank (DEC-014)

**Kenya 2024 PAYE Tax Bands:**
```
0 – 24,000        → 10%
24,001 – 32,333   → 25%
32,334 – 40,667   → 30%
40,668 – 60,000   → 32.5%
60,001+            → 35%
Personal relief: Ksh 2,400/month
```

**NHIF:** Based on gross salary bracket (Ksh 150 – Ksh 1,700/month)  
**NSSF:** 6% of gross, max Ksh 2,160/month (Tier I + II)

**Data:**
```
StaffSalaryRecord { id, schoolId, userId, basicPay, allowances Json, deductions Json, salaryGrade, effectiveFrom }
SalaryRun { id, schoolId, month, year, status (DRAFT|APPROVED|DISBURSED), approvedByUserId, createdAt }
SalaryRunItem { id, salaryRunId, schoolId, userId, grossPay, paye, nhif, nssf, otherDeductions, netPay }
Payslip { id, salaryRunItemId, schoolId, userId, s3Key, sentAt }
```

---

## FR-008: KRA Tax Reports (Manual Export)

**Priority:** P0 | **Roles:** Accountant generates

Per DEC-009 — generate formatted reports for manual iTax upload, no direct API.

| Report | Format | Period |
|---|---|---|
| PAYE Monthly Return (P10) | CSV (iTax format) | Monthly |
| VAT Return (VAT3) | CSV | Monthly (if VAT-registered) |
| Withholding Tax Return (WHT) | CSV | Monthly |

All reports: clearly labelled "For professional review before iTax submission"

**Data:**
```
TaxRecord { id, schoolId, taxType (PAYE|VAT|WHT), period, amount, status (DRAFT|FILED), s3Key, createdAt }
```

---

## FR-009: Bank Statement Import & Reconciliation

**Priority:** P1 | **Roles:** Accountant

- Accountant uploads bank statement (CSV or Excel)
- System parses columns: date, description, debit, credit, balance
- System attempts to match transactions to `FeePayment` records by amount + date
- Matched: automatically reconciled
- Unmatched: shown in "Pending Reconciliation" list for manual matching
- Reconciliation report downloadable as PDF

**Data:**
```
BankStatement { id, schoolId, bankName, accountNo, uploadedByUserId, fileName, s3Key, fromDate, toDate, createdAt }
BankTransaction { id, bankStatementId, schoolId, transactionDate, description, debit, credit, balance, matchedFeePaymentId?, status (UNMATCHED|MATCHED|IGNORED) }
```

---

## FR-010: School Subscription Billing

**Priority:** P0 | **Roles:** Super Admin manages; Headteacher views own subscription

**Stripe flow (international/card):**
1. Super Admin or Headteacher opens billing page → clicks "Subscribe / Upgrade"
2. Stripe Checkout session created → user redirected
3. Payment completed → Stripe webhook → `subscription.updated`
4. `SchoolSubscription` updated to ACTIVE + tier

**Mpesa flow (Kenyan schools):**
1. Headteacher opens billing → clicks "Renew via Mpesa" → STK Push initiated
2. School's registered number pays EduManage paybill
3. Callback processed → subscription renewed for next term

**Subscription states:** TRIAL → ACTIVE → PAST_DUE (14 days) → SUSPENDED

**Data:**
```
SchoolSubscription { id, schoolId, tier, status, stripeSubscriptionId?, stripeCustomerId?,
  currentPeriodStart, currentPeriodEnd, trialEndsAt?, cancelledAt? }
```

---

## Non-Functional Requirements

- Mpesa STK Push response: <3 seconds to confirmation screen
- Fee statement page: <1 second load (cached StudentFeeBalance)
- PDF generation: <5 seconds (server-side via Puppeteer or `@react-pdf/renderer`)
- All financial data: immutable audit trail (AuditLog for every mutation)
- Mpesa callbacks: idempotent — duplicate receipt number silently ignored
- Payroll PAYE calculation: unit-tested with Kenya Revenue Authority published examples

---

## Dashboard Tabs Delivered

**Accountant:** Dashboard, Payments, School Fees, Payroll, Tax, Bank, Subscriptions, Reports, Budgets, Calendar  
**Headteacher:** Financial Overview, Salary Approval, Budget Approvals, Subscription Status  
**Parent:** Fee Statement, Pay Now (STK Push), Transaction History, Download Statement  
**Super Admin:** All school subscriptions, billing status, revenue overview
