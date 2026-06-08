# Builder Handoff — Sprint 004: Financial Management & Mpesa

Paste everything below into Claude Code. Sprint 003 must be passing.

---

You are building Sprint 004 of EduManage: the financial layer that enables first revenue.

## Pre-read (mandatory, in order):
1. `AGENTS.md` + `CONTEXT.md`
2. `planning/DECISIONS.md` (DEC-006 Mpesa+Stripe, DEC-009 KRA, DEC-014 Salary)
3. `planning/RISKS.md` (RISK-001 Mpesa reliability, RISK-003 KRA complexity)
4. `planning/sprints/004-financial-mpesa/requirements.md`
5. `planning/sprints/004-financial-mpesa/blueprint.md`
6. `planning/sprints/004-financial-mpesa/acceptance.md`

## Agent routing:
- Prisma models, tRPC routers, webhook handlers → `nextjs-expert`
- Mpesa/Stripe webhook logic, Inngest jobs, CI → `devops`
- Fee payment UI, payslip viewer, financial charts → `react-frontend`
- Role guards on financial procedures → `clerk-auth`

## Summarise before coding — output:
1. New files (full paths)
2. Modified files
3. New Prisma migration name
4. New Inngest job names
5. New env vars needed
6. Any `// ARCH-QUESTION:` items

Wait for approval, then build.

## Critical rules:
- Mpesa callback handler MUST return 200 in <5 seconds — use Inngest, never process synchronously
- ALL Mpesa receipt numbers: stored with `@unique` constraint — database-level idempotency
- FeePayment with `status = COMPLETED`: server MUST reject any update to `amount` or `status`
- PAYE calculation: pure function in `lib/financial/paye.ts` — unit tested before UI
- PDF generation: Inngest job only — never block an HTTP request with PDF rendering
- Stripe webhook: ALWAYS verify signature with `stripe.webhooks.constructEvent()`
- Financial tRPC procedures: Accountant-only mutations, read access by Headteacher/Parent (own data)

## Install packages:
```bash
npm install @react-pdf/renderer xlsx papaparse stripe
npm install -D @types/papaparse
```

## Implementation sequence:
1. Prisma migration: `sprint_004_financial`
2. RLS policies for all new financial tables
3. `lib/financial/paye.ts` + unit tests (BEFORE any UI)
4. `lib/mpesa/daraja.ts` — STK Push client
5. `lib/stripe/index.ts` — Stripe client + checkout session
6. `server/routers/financial.ts` — all procedures
7. `/api/webhooks/mpesa/route.ts`
8. `/api/webhooks/stripe/route.ts`
9. Inngest jobs: processMpesaPayment, generatePayslips, feeReminder, subscriptionExpiry, bankReconcile, generateFinancialReport
10. Fee statement page (Parent + Accountant views)
11. Payment initiation UI (Parent: STK Push modal)
12. Accountant dashboard: payments list, fee structure config, payroll, reports, tax, bank
13. Integration tests: Mpesa idempotency, Stripe webhook, PAYE calculation

## Definition of done: all AC-001 through AC-009 pass.
