# Blueprint — Sprint 004: Financial Management & Mpesa

**Project:** EduManage  
**Sprint:** 004  
**Status:** Approved  
**Last updated:** 2026-05-29

---

## Architecture Additions

### New Prisma Models

```prisma
// ─── Fee Management ──────────────────────────────────────────────

model FeeStructure {
  id               String     @id @default(uuid())
  schoolId         String
  termId           String
  classId          String
  approvedByUserId String?
  isActive         Boolean    @default(false)
  createdAt        DateTime   @default(now())
  items            FeeItem[]
  @@unique([schoolId, termId, classId])
  @@index([schoolId])
}

model FeeItem {
  id              String       @id @default(uuid())
  feeStructureId  String
  schoolId        String
  name            String
  amount          Decimal      @db.Decimal(12, 2)
  isMandatory     Boolean      @default(true)
  dueDate         DateTime?
  category        FeeCategory  // TUITION | ACTIVITY | TRANSPORT | MEALS | BOARDING | LIBRARY | OTHER
  structure       FeeStructure @relation(fields: [feeStructureId], references: [id])
  @@index([schoolId])
}

model StudentFeeBalance {
  id         String   @id @default(uuid())
  schoolId   String
  studentId  String
  termId     String
  totalOwed  Decimal  @db.Decimal(12, 2) @default(0)
  totalPaid  Decimal  @db.Decimal(12, 2) @default(0)
  balance    Decimal  @db.Decimal(12, 2) @default(0)  // = totalOwed - totalPaid
  updatedAt  DateTime @updatedAt
  @@unique([schoolId, studentId, termId])
  @@index([schoolId, termId])
}

model FeePayment {
  id                   String        @id @default(uuid())
  schoolId             String
  studentId            String
  termId               String
  feeItemId            String?
  amount               Decimal       @db.Decimal(12, 2)
  currency             String        @default("KES")
  paymentMethod        PaymentMethod // MPESA | BANK | CASH | CHEQUE
  mpesaReceiptNo       String?       @unique
  mpesaPhoneNo         String?
  mpesaTransactionDate DateTime?
  stripePaymentId      String?
  reference            String?
  status               PaymentStatus @default(PENDING)
  recordedByUserId     String?       // null = system (Mpesa callback)
  paidAt               DateTime?
  createdAt            DateTime      @default(now())
  @@index([schoolId, studentId])
  @@index([schoolId, termId])
}

model MpesaStkRequest {
  id                String   @id @default(uuid())
  schoolId          String
  checkoutRequestId String   @unique
  merchantRequestId String
  studentId         String
  amount            Decimal  @db.Decimal(12, 2)
  phoneNo           String
  status            String   @default("PENDING")
  resultCode        Int?
  resultDesc        String?
  createdAt         DateTime @default(now())
  @@index([schoolId])
}

enum FeeCategory   { TUITION ACTIVITY TRANSPORT MEALS BOARDING LIBRARY OTHER }
enum PaymentMethod { MPESA BANK CASH CHEQUE }
enum PaymentStatus { PENDING COMPLETED FAILED REVERSED }

// ─── Payroll ─────────────────────────────────────────────────────

model StaffSalaryRecord {
  id             String   @id @default(uuid())
  schoolId       String
  userId         String
  basicPay       Decimal  @db.Decimal(12, 2)
  allowances     Json     // { housing: 5000, transport: 3000, ... }
  salaryGrade    String?
  effectiveFrom  DateTime @db.Date
  @@index([schoolId, userId])
}

model SalaryRun {
  id               String       @id @default(uuid())
  schoolId         String
  month            Int          // 1-12
  year             Int
  status           SalaryStatus @default(DRAFT)
  approvedByUserId String?
  approvedAt       DateTime?
  createdAt        DateTime     @default(now())
  items            SalaryRunItem[]
  @@unique([schoolId, month, year])
  @@index([schoolId])
}

model SalaryRunItem {
  id              String    @id @default(uuid())
  salaryRunId     String
  schoolId        String
  userId          String
  grossPay        Decimal   @db.Decimal(12, 2)
  paye            Decimal   @db.Decimal(12, 2)
  nhif            Decimal   @db.Decimal(12, 2)
  nssf            Decimal   @db.Decimal(12, 2)
  otherDeductions Decimal   @db.Decimal(12, 2) @default(0)
  netPay          Decimal   @db.Decimal(12, 2)
  salaryRun       SalaryRun @relation(fields: [salaryRunId], references: [id])
  payslip         Payslip?
  @@index([schoolId, userId])
}

model Payslip {
  id              String        @id @default(uuid())
  salaryRunItemId String        @unique
  schoolId        String
  userId          String
  s3Key           String
  sentAt          DateTime?
  item            SalaryRunItem @relation(fields: [salaryRunItemId], references: [id])
  @@index([schoolId, userId])
}

enum SalaryStatus { DRAFT APPROVED DISBURSED }

// ─── Tax Records ─────────────────────────────────────────────────

model TaxRecord {
  id              String    @id @default(uuid())
  schoolId        String
  taxType         TaxType   // PAYE | VAT | WHT
  period          String    // "2026-05" (YYYY-MM)
  amount          Decimal   @db.Decimal(12, 2)
  status          TaxStatus @default(DRAFT)
  s3Key           String?
  generatedByUserId String
  createdAt       DateTime  @default(now())
  @@unique([schoolId, taxType, period])
  @@index([schoolId])
}

enum TaxType   { PAYE VAT WHT }
enum TaxStatus { DRAFT REVIEWED FILED }

// ─── Bank Statement ───────────────────────────────────────────────

model BankStatement {
  id                String            @id @default(uuid())
  schoolId          String
  bankName          String
  accountNo         String
  uploadedByUserId  String
  fileName          String
  s3Key             String
  fromDate          DateTime          @db.Date
  toDate            DateTime          @db.Date
  createdAt         DateTime          @default(now())
  transactions      BankTransaction[]
  @@index([schoolId])
}

model BankTransaction {
  id                    String          @id @default(uuid())
  bankStatementId       String
  schoolId              String
  transactionDate       DateTime        @db.Date
  description           String
  debit                 Decimal?        @db.Decimal(12, 2)
  credit                Decimal?        @db.Decimal(12, 2)
  balance               Decimal?        @db.Decimal(12, 2)
  matchedFeePaymentId   String?
  status                ReconcileStatus @default(UNMATCHED)
  statement             BankStatement   @relation(fields: [bankStatementId], references: [id])
  @@index([schoolId])
}

enum ReconcileStatus { UNMATCHED MATCHED IGNORED }

// ─── Subscription ────────────────────────────────────────────────

model SchoolSubscription {
  id                   String             @id @default(uuid())
  schoolId             String             @unique
  tier                 SubscriptionTier
  status               SubscriptionStatus
  stripeSubscriptionId String?
  stripeCustomerId     String?
  currentPeriodStart   DateTime?
  currentPeriodEnd     DateTime?
  trialEndsAt          DateTime?
  cancelledAt          DateTime?
  updatedAt            DateTime           @updatedAt
  @@index([schoolId])
}

// ─── Financial Report Log ─────────────────────────────────────────

model FinancialReport {
  id                String     @id @default(uuid())
  schoolId          String
  reportType        String
  parameters        Json
  generatedByUserId String
  s3Key             String
  createdAt         DateTime   @default(now())
  @@index([schoolId])
}
```

---

## tRPC Router: `financial.*`

```typescript
// Procedures (role guards omitted for brevity — see DOMAIN.md)

financial.fee.configureStructure({ termId, classId, items[] })         // ACCOUNTANT
financial.fee.approveStructure({ feeStructureId })                      // HEADTEACHER
financial.fee.getStudentStatement({ studentId, termId })                 // ACCOUNTANT | PARENT(own) | STUDENT(own)
financial.fee.initiatePayment({ studentId, termId, amount, phoneNo })   // PARENT(own) | ACCOUNTANT
financial.fee.recordManualPayment({ studentId, amount, method, ref })   // ACCOUNTANT
financial.fee.downloadStatement({ studentId?, termId, from, to, format }) // ACCOUNTANT | PARENT(own)

financial.payroll.createRun({ month, year })                             // ACCOUNTANT
financial.payroll.enterSalaries({ salaryRunId, items[] })               // ACCOUNTANT
financial.payroll.calculatePAYE({ grossPay })                           // server utility
financial.payroll.approveRun({ salaryRunId })                           // HEADTEACHER
financial.payroll.generatePayslips({ salaryRunId })                     // ACCOUNTANT (triggers Inngest)

financial.tax.generatePAYE({ month, year })                             // ACCOUNTANT
financial.tax.generateVAT({ month, year })                              // ACCOUNTANT
financial.tax.downloadReport({ taxRecordId })                           // ACCOUNTANT

financial.bank.uploadStatement({ file })                                 // ACCOUNTANT
financial.bank.reconcile({ statementId })                               // ACCOUNTANT (triggers Inngest)
financial.bank.manualMatch({ transactionId, feePaymentId })             // ACCOUNTANT

financial.reports.generate({ reportType, parameters })                   // ACCOUNTANT | HEADTEACHER
financial.reports.download({ reportId })                                 // ACCOUNTANT | HEADTEACHER

financial.subscription.getStatus()                                       // HEADTEACHER | SUPER_ADMIN
financial.subscription.initiateStripeCheckout({ tier })                  // HEADTEACHER | SUPER_ADMIN
financial.subscription.initiateRenewalMpesa({ phoneNo })                 // HEADTEACHER
```

---

## Webhook Handlers

### `/api/webhooks/mpesa`

```typescript
// Validate Safaricom origin IP + credentials
// Parse: ResultCode, ResultDesc, MpesaReceiptNumber, Amount, TransactionDate, PhoneNumber
// Emit Inngest event: mpesa/callback.received
// Return 200 OK immediately (Safaricom times out at 5s)

export async function POST(req: Request) {
  const body = await req.json()
  
  // Verify IP from Safaricom whitelist
  const ip = req.headers.get('x-forwarded-for')
  if (!SAFARICOM_IPS.includes(ip)) return new Response('Forbidden', { status: 403 })
  
  await inngest.send({
    name: 'mpesa/callback.received',
    data: {
      checkoutRequestId: body.Body.stkCallback.CheckoutRequestID,
      resultCode: body.Body.stkCallback.ResultCode,
      resultDesc: body.Body.stkCallback.ResultDesc,
      callbackMetadata: body.Body.stkCallback.CallbackMetadata,
    }
  })
  
  return Response.json({ ResultCode: 0, ResultDesc: 'Accepted' })
}
```

### `/api/webhooks/stripe`

```typescript
// Verify Stripe signature using STRIPE_WEBHOOK_SECRET
// Handle: checkout.session.completed, customer.subscription.updated, customer.subscription.deleted, invoice.payment_failed
// Emit Inngest event: stripe/webhook.received
```

---

## Inngest Jobs

| Job | Trigger | Action |
|---|---|---|
| `processMpesaPayment` | `mpesa/callback.received` | Idempotent: check receipt exists → create FeePayment → update StudentFeeBalance → generate receipt PDF → email parent |
| `generatePayslips` | `payroll/run.approved` | For each SalaryRunItem: generate PDF payslip (S3) → email staff via Resend |
| `feeReminder` | Cron: every Monday 08:00 KE | Find outstanding balances → send in-app + email reminders |
| `subscriptionExpiry` | Cron: daily 06:00 | Check subscriptions → flag PAST_DUE after 1 day → SUSPENDED after 14 days |
| `bankReconcile` | `bank/statement.uploaded` | Parse bank CSV → attempt auto-match against FeePayments → save BankTransactions |
| `generateFinancialReport` | `reports/generate.requested` | Query DB → generate PDF/Excel → save to S3 → notify user |
| `stripeWebhook` | `stripe/webhook.received` | Update SchoolSubscription status/tier |

---

## PAYE Calculation (Pure Function — Unit Tested)

```typescript
// lib/financial/paye.ts
export function calculateKenyaPAYE(grossMonthly: number): {
  taxableIncome: number
  paye: number
  nhif: number
  nssf: number
  netPay: number
} {
  // Kenya 2024 bands
  let paye = 0
  const bands = [
    { limit: 24000, rate: 0.10 },
    { limit: 8333,  rate: 0.25 },
    { limit: 8334,  rate: 0.30 },
    { limit: 19333, rate: 0.325 },
    { limit: Infinity, rate: 0.35 },
  ]
  let remaining = grossMonthly
  for (const band of bands) {
    const taxable = Math.min(remaining, band.limit)
    paye += taxable * band.rate
    remaining -= taxable
    if (remaining <= 0) break
  }
  paye = Math.max(0, paye - 2400) // personal relief

  // NHIF (bracket-based, simplified to common values)
  const nhif = grossMonthly <= 5999 ? 150
    : grossMonthly <= 7999 ? 300
    : grossMonthly <= 11999 ? 400
    : grossMonthly <= 14999 ? 500
    : grossMonthly <= 19999 ? 600
    : grossMonthly <= 24999 ? 750
    : grossMonthly <= 29999 ? 850
    : grossMonthly <= 34999 ? 900
    : grossMonthly <= 39999 ? 950
    : grossMonthly <= 44999 ? 1000
    : grossMonthly <= 49999 ? 1100
    : grossMonthly <= 59999 ? 1200
    : grossMonthly <= 69999 ? 1300
    : grossMonthly <= 79999 ? 1400
    : grossMonthly <= 89999 ? 1500
    : grossMonthly <= 99999 ? 1600 : 1700

  // NSSF Tier I (6% of first 6,000) + Tier II (6% of next 12,000), max 2,160
  const nssf = Math.min(grossMonthly * 0.06, 2160)

  return {
    taxableIncome: grossMonthly,
    paye: Math.round(paye),
    nhif,
    nssf: Math.round(nssf),
    netPay: Math.round(grossMonthly - paye - nhif - nssf),
  }
}
```

---

## PDF Generation Strategy

Use `@react-pdf/renderer` for server-side PDF generation in Inngest jobs:
- Payslips: school logo, staff name, salary breakdown, net pay, month/year
- Fee receipts: school logo, student name, amount, receipt no, date
- Financial reports: tables, charts (convert Recharts to SVG → embed in PDF)
- All PDFs: uploaded to S3, presigned URL returned to client for download

---

## Security Controls (Financial-specific)

1. All `FeePayment` records: immutable after `status = COMPLETED` — no UPDATE allowed
2. Payroll approval: 2-step — Accountant prepares, Headteacher approves — no single person can do both
3. Mpesa callback: IP whitelist + credential check before processing
4. Stripe webhook: signature verification mandatory
5. Financial reports: generated async (Inngest) — never expose raw DB to PDF renderer on request thread
6. KRA tax reports: marked "DRAFT — not for filing" until Accountant explicitly marks as REVIEWED

---

## Environment Variables (new this sprint)

```env
# Mpesa Daraja
MPESA_CONSUMER_KEY=
MPESA_CONSUMER_SECRET=
MPESA_PASSKEY=
MPESA_SHORTCODE=           # School's paybill or till number
MPESA_EDUMANAGE_SHORTCODE= # EduManage's own paybill for subscriptions
MPESA_ENVIRONMENT=sandbox  # sandbox | production
MPESA_CALLBACK_URL=https://edumanage.app/api/webhooks/mpesa

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
STRIPE_PRICE_ID_BASIC=
STRIPE_PRICE_ID_STANDARD=
STRIPE_PRICE_ID_PREMIUM=
```

---

## New npm Packages

```bash
npm install @react-pdf/renderer xlsx papaparse
npm install -D @types/papaparse
```
