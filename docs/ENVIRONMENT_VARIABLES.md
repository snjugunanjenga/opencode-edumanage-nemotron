# ENVIRONMENT_VARIABLES.md — EduManage

**Last updated:** 2026-05-29  
**Source of truth:** Vercel Environment Variables dashboard  
**Never commit secrets to git.**

---

## Variable Catalog

### Database (Neon)

| Variable | Required | Env | Description |
|---|---|---|---|
| `DATABASE_URL` | ✅ | All | Neon pooled connection string (Prisma runtime) |
| `DIRECT_URL` | ✅ | All | Neon direct connection (Prisma migrations only) |

```env
# Format: postgres://user:password@host/dbname?sslmode=require
DATABASE_URL=postgres://...@ep-xxx.eu-west-2.aws.neon.tech/edumanage?sslmode=require&pgbouncer=true
DIRECT_URL=postgres://...@ep-xxx.eu-west-2.aws.neon.tech/edumanage?sslmode=require
```

---

### Authentication (Clerk)

| Variable | Required | Env | Description |
|---|---|---|---|
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | ✅ | All | Clerk publishable key (browser-safe) |
| `CLERK_SECRET_KEY` | ✅ | All | Clerk secret key (server only) |
| `CLERK_WEBHOOK_SECRET` | ✅ | All | Svix webhook signing secret |
| `NEXT_PUBLIC_CLERK_SIGN_IN_URL` | ✅ | All | `/sign-in` |
| `NEXT_PUBLIC_CLERK_SIGN_UP_URL` | ✅ | All | `/sign-up` (invite-only — redirects to error) |
| `NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL` | ✅ | All | `/dashboard` |
| `NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL` | ✅ | All | `/onboarding` |

---

### Payments — Mpesa (Daraja)

| Variable | Required | Env | Description |
|---|---|---|---|
| `MPESA_CONSUMER_KEY` | ✅ | All | Daraja API consumer key |
| `MPESA_CONSUMER_SECRET` | ✅ | All | Daraja API consumer secret |
| `MPESA_PASSKEY` | ✅ | All | STK Push passkey |
| `MPESA_SHORTCODE` | ✅ | All | School's paybill/till number |
| `MPESA_EDUMANAGE_SHORTCODE` | ✅ | All | EduManage's own paybill (subscription payments) |
| `MPESA_ENVIRONMENT` | ✅ | All | `sandbox` or `production` |
| `MPESA_CALLBACK_URL` | ✅ | Prod | `https://edumanage.app/api/webhooks/mpesa` |

```env
MPESA_CONSUMER_KEY=xxx
MPESA_CONSUMER_SECRET=xxx
MPESA_PASSKEY=xxx
MPESA_SHORTCODE=174379         # sandbox default
MPESA_EDUMANAGE_SHORTCODE=xxx
MPESA_ENVIRONMENT=sandbox
MPESA_CALLBACK_URL=https://yourdomain.vercel.app/api/webhooks/mpesa
```

---

### Payments — Stripe

| Variable | Required | Env | Description |
|---|---|---|---|
| `STRIPE_SECRET_KEY` | ✅ | All | Stripe secret key (`sk_test_...` / `sk_live_...`) |
| `STRIPE_WEBHOOK_SECRET` | ✅ | All | Stripe webhook signing secret (`whsec_...`) |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | ✅ | All | Stripe publishable key (`pk_test_...`) |
| `STRIPE_PRICE_ID_BASIC` | ✅ | All | Stripe Price ID for Basic tier |
| `STRIPE_PRICE_ID_STANDARD` | ✅ | All | Stripe Price ID for Standard tier |
| `STRIPE_PRICE_ID_PREMIUM` | ✅ | All | Stripe Price ID for Premium tier |

---

### File Storage (AWS S3)

| Variable | Required | Env | Description |
|---|---|---|---|
| `AWS_REGION` | ✅ | All | `eu-west-1` |
| `AWS_ACCESS_KEY_ID` | ✅ | All | IAM user access key (least-privilege) |
| `AWS_SECRET_ACCESS_KEY` | ✅ | All | IAM user secret key |
| `AWS_S3_BUCKET` | ✅ | All | Bucket name (`edumanage-prod-assets` / `edumanage-staging-assets`) |

```env
AWS_REGION=eu-west-1
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=xxx
AWS_S3_BUCKET=edumanage-prod-assets
```

---

### AI (Google Gemini)

| Variable | Required | Env | Description |
|---|---|---|---|
| `GOOGLE_GENERATIVE_AI_API_KEY` | ✅ | All | Google AI Studio API key (Sprint 007+) |

---

### Email (Resend)

| Variable | Required | Env | Description |
|---|---|---|---|
| `RESEND_API_KEY` | ✅ | All | Resend API key |
| `RESEND_FROM_EMAIL` | ✅ | All | `noreply@edumanage.app` |
| `RESEND_FROM_NAME` | Optional | All | `EduManage` |

---

### Async Jobs (Inngest)

| Variable | Required | Env | Description |
|---|---|---|---|
| `INNGEST_EVENT_KEY` | ✅ | All | Inngest event signing key |
| `INNGEST_SIGNING_KEY` | ✅ | All | Inngest function signing key |

```env
INNGEST_EVENT_KEY=signkey-prod-xxx
INNGEST_SIGNING_KEY=signkey-prod-xxx
```

---

### Monitoring

| Variable | Required | Env | Description |
|---|---|---|---|
| `SENTRY_DSN` | ✅ | Prod | Sentry server-side DSN |
| `NEXT_PUBLIC_SENTRY_DSN` | ✅ | Prod | Sentry client-side DSN |
| `SENTRY_AUTH_TOKEN` | CI only | Prod | For source map upload in CI |

---

### Application

| Variable | Required | Env | Description |
|---|---|---|---|
| `NEXT_PUBLIC_APP_URL` | ✅ | All | `https://edumanage.app` (prod) / `https://staging.edumanage.app` |
| `NODE_ENV` | Auto | All | Set by Vercel automatically |

---

## Environment Setup by Context

### Local Development (`.env.local`)
```env
# Database — Neon dev branch
DATABASE_URL=postgres://...neon.tech/edumanage_dev?sslmode=require&pgbouncer=true
DIRECT_URL=postgres://...neon.tech/edumanage_dev?sslmode=require

# Auth — Clerk test keys
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
CLERK_WEBHOOK_SECRET=whsec_test_...
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/dashboard
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/onboarding

# Mpesa — Daraja sandbox
MPESA_ENVIRONMENT=sandbox
MPESA_SHORTCODE=174379
MPESA_CONSUMER_KEY=xxx
MPESA_CONSUMER_SECRET=xxx
MPESA_PASSKEY=bfb279f9aa9bdbcf158e97dd71a467cd2e0c893059b10f78e6b72ada1ed2c919
MPESA_CALLBACK_URL=https://YOUR_NGROK_URL/api/webhooks/mpesa

# Stripe — test mode
STRIPE_SECRET_KEY=sk_test_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_test_...
STRIPE_PRICE_ID_BASIC=price_test_...
STRIPE_PRICE_ID_STANDARD=price_test_...
STRIPE_PRICE_ID_PREMIUM=price_test_...

# AWS S3 — use staging bucket locally
AWS_REGION=eu-west-1
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_S3_BUCKET=edumanage-staging-assets

# Gemini
GOOGLE_GENERATIVE_AI_API_KEY=...

# Email — Resend test mode
RESEND_API_KEY=re_test_...
RESEND_FROM_EMAIL=onboarding@resend.dev

# Inngest — local dev mode (inngest-cli)
INNGEST_EVENT_KEY=local
INNGEST_SIGNING_KEY=local

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

### CI/CD (GitHub Secrets)
All production variables stored as GitHub repository secrets under `Settings → Secrets → Actions`. Named identically to the variables above.

Additional CI-only variables:
```env
NEON_BRANCH_URL=    # Set by Neon GitHub integration for PR branches
CLERK_TEST_SECRET_KEY=sk_test_...
SNYK_TOKEN=         # For security scanning
SENTRY_AUTH_TOKEN=  # For source map upload
```

### Production (Vercel Environment Variables)
Set via Vercel dashboard → Project → Settings → Environment Variables.  
Select "Production" scope for production values, "Preview" for staging.

---

## Variable Checklist Per Sprint

| Variable Group | First Needed |
|---|---|
| DATABASE_URL, DIRECT_URL | Sprint 002 |
| Clerk variables | Sprint 002 |
| AWS S3 variables | Sprint 003 (file uploads) |
| Mpesa variables | Sprint 004 |
| Stripe variables | Sprint 004 |
| Resend variables | Sprint 004 (payslip emails) |
| Inngest variables | Sprint 003 (first jobs) |
| Google Gemini key | Sprint 007 |
| Sentry DSN | Sprint 004 (production-ready) |

---

## Security Notes

- **Never log** `CLERK_SECRET_KEY`, `MPESA_CONSUMER_SECRET`, `AWS_SECRET_ACCESS_KEY`, `STRIPE_SECRET_KEY`, `GOOGLE_GENERATIVE_AI_API_KEY`
- Rotate all secrets quarterly and after any team member departure
- IAM user (`edumanage-app`) has S3 permissions scoped to `edumanage-prod-assets/*` only
- Daraja production credentials require Safaricom business KYC — allow 2–4 weeks
- Stripe live mode requires business verification — allow 1–3 business days
