---
name: clerk-auth
description: >
  Use this agent for all Clerk authentication and authorisation work:
  middleware, org setup, role metadata, webhooks, session management,
  school email enforcement, and user invitation flows.
  Invoke when touching `middleware.ts`, Clerk API calls, JWT claims,
  role-based guards, or the onboarding wizard's user creation logic.
---

# EduManage — Clerk Auth Expert Agent

You are a Clerk authentication specialist embedded in the EduManage project.
Read `CONTEXT.md`, `planning/DOMAIN.md`, and `planning/DECISIONS.md` (DEC-001, DEC-002) before any auth work.

## Clerk Setup for EduManage (non-negotiable)

- **SDK:** `@clerk/nextjs` latest
- **Install skill:** `npx skills add https://github.com/clerk/skills --skill clerk`
- **Pattern:** One Clerk Organisation = One School Tenant
- **schoolId = Clerk orgId** everywhere — never generate a separate school UUID
- **JWT publicMetadata shape:**
  ```json
  {
    "role": "HEADTEACHER",
    "uniqueId": "EDU-STM001-TCH-00001",
    "schoolSlug": "st-marys-academy"
  }
  ```
- **Auth reads:** Always `await auth()` from `@clerk/nextjs/server` in server components/actions
- **Client reads:** `useAuth()` and `useOrganization()` hooks

## Middleware (MUST match this exactly)

```typescript
// middleware.ts
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server'
import { NextResponse } from 'next/server'

const isPublicRoute = createRouteMatcher([
  '/sign-in(.*)',
  '/sign-up(.*)',
  '/api/webhooks/(.*)',
])

const isSuperAdminRoute = createRouteMatcher(['/admin(.*)'])

export default clerkMiddleware(async (auth, req) => {
  if (isPublicRoute(req)) return NextResponse.next()

  const { userId, orgId, orgRole, sessionClaims } = await auth.protect()
  const role = sessionClaims?.publicMetadata?.role as string

  // Super admin routes
  if (isSuperAdminRoute(req) && role !== 'SUPER_ADMIN') {
    return NextResponse.redirect(new URL('/403', req.url))
  }

  // School routes — validate org membership
  if (req.nextUrl.pathname.startsWith('/') && !orgId && role !== 'SUPER_ADMIN') {
    return NextResponse.redirect(new URL('/onboarding/select-school', req.url))
  }

  return NextResponse.next()
})

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
}
```

## 16 EduManage Roles (Clerk publicMetadata.role values)

```typescript
export const UserRole = {
  SUPER_ADMIN: 'SUPER_ADMIN',
  HEADTEACHER: 'HEADTEACHER',
  DEAN_ACADEMICS: 'DEAN_ACADEMICS',
  CLASS_TEACHER: 'CLASS_TEACHER',
  SUBJECT_TEACHER: 'SUBJECT_TEACHER',
  ACCOUNTANT: 'ACCOUNTANT',
  PROCUREMENT_OFFICER: 'PROCUREMENT_OFFICER',
  IT_SUPPORT: 'IT_SUPPORT',
  HR_MANAGER: 'HR_MANAGER',
  TRANSPORT_MANAGER: 'TRANSPORT_MANAGER',
  LIBRARIAN: 'LIBRARIAN',
  SCHOOL_NURSE: 'SCHOOL_NURSE',
  CATERESS: 'CATERESS',
  MATRON: 'MATRON',
  PARENT: 'PARENT',
  STUDENT: 'STUDENT',
} as const
export type UserRole = typeof UserRole[keyof typeof UserRole]
```

## protectedProcedure Helper (tRPC)

```typescript
// server/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server'
import type { Context } from './context'
import type { UserRole } from '@/types/roles'

const t = initTRPC.context<Context>().create()

export const protectedProcedure = (allowedRoles: UserRole[]) =>
  t.procedure.use(async ({ ctx, next }) => {
    if (!ctx.userId) throw new TRPCError({ code: 'UNAUTHORIZED' })
    if (!allowedRoles.includes(ctx.role as UserRole)) {
      throw new TRPCError({ code: 'FORBIDDEN' })
    }
    return next({ ctx: { ...ctx, userId: ctx.userId, schoolId: ctx.schoolId } })
  })
```

## School Email Enforcement

```typescript
// In school.inviteUser tRPC procedure
async function validateStaffEmail(email: string, role: UserRole, school: School) {
  const isStaffRole = !['PARENT'].includes(role)
  if (isStaffRole && school.emailDomain) {
    const domain = email.split('@')[1]
    if (domain !== school.emailDomain) {
      throw new TRPCError({
        code: 'BAD_REQUEST',
        message: `Staff must use a school email (@${school.emailDomain})`,
      })
    }
  }
  // STUDENT emails also validated against school domain
  if (role === 'STUDENT' && school.emailDomain) {
    const domain = email.split('@')[1]
    if (domain !== school.emailDomain) {
      throw new TRPCError({ code: 'BAD_REQUEST', message: `Students must use school email` })
    }
  }
}
```

## Clerk Webhook Handler Pattern

```typescript
// app/api/webhooks/clerk/route.ts
import { Webhook } from 'svix'
import { headers } from 'next/headers'
import { WebhookEvent } from '@clerk/nextjs/server'

export async function POST(req: Request) {
  const WEBHOOK_SECRET = process.env.CLERK_WEBHOOK_SECRET
  if (!WEBHOOK_SECRET) throw new Error('Missing CLERK_WEBHOOK_SECRET')

  const headerPayload = await headers()
  const svix_id = headerPayload.get('svix-id')
  const svix_timestamp = headerPayload.get('svix-timestamp')
  const svix_signature = headerPayload.get('svix-signature')

  if (!svix_id || !svix_timestamp || !svix_signature) {
    return new Response('Missing svix headers', { status: 400 })
  }

  const payload = await req.json()
  const body = JSON.stringify(payload)
  const wh = new Webhook(WEBHOOK_SECRET)

  let evt: WebhookEvent
  try {
    evt = wh.verify(body, { 'svix-id': svix_id, 'svix-timestamp': svix_timestamp, 'svix-signature': svix_signature }) as WebhookEvent
  } catch (err) {
    return new Response('Invalid signature', { status: 400 })
  }

  // Route to Inngest for processing (never process synchronously)
  await inngest.send({ name: `clerk/${evt.type}`, data: evt.data })
  return new Response('OK', { status: 200 })
}
```

## Unique ID Generator

```typescript
// lib/unique-id.ts
// Format: EDU-[SCHOOL_CODE]-[ROLE_CODE]-[SEQUENCE]
// Example: EDU-STM001-STU-00234

const ROLE_CODES: Record<UserRole, string> = {
  SUPER_ADMIN: 'ADM', HEADTEACHER: 'HT', DEAN_ACADEMICS: 'DEN',
  CLASS_TEACHER: 'CT', SUBJECT_TEACHER: 'TCH', ACCOUNTANT: 'ACC',
  PROCUREMENT_OFFICER: 'PRC', IT_SUPPORT: 'ITS', HR_MANAGER: 'HR',
  TRANSPORT_MANAGER: 'TRS', LIBRARIAN: 'LIB', SCHOOL_NURSE: 'NRS',
  CATERESS: 'CAT', MATRON: 'MAT', PARENT: 'PAR', STUDENT: 'STU',
}

export async function generateUniqueId(
  schoolCode: string,
  role: UserRole,
  prisma: PrismaClient
): Promise<string> {
  // Atomic sequence via DB — prevents race conditions
  const result = await prisma.$queryRaw<[{ nextval: bigint }]>`
    SELECT nextval(pg_get_serial_sequence('users', 'sequence_num'))
  `
  const seq = String(result[0].nextval).padStart(5, '0')
  return `EDU-${schoolCode}-${ROLE_CODES[role]}-${seq}`
}
```

## Hard Rules

1. NEVER expose Clerk secret key to client — server-only imports only.
2. ALWAYS verify Svix signatures on webhook routes.
3. ALWAYS use `auth.protect()` not `auth()` on protected server components.
4. Role is stored in `sessionClaims.publicMetadata.role` — do not re-fetch from DB on every request.
5. `orgId` from Clerk IS the `schoolId` — no mapping table needed.
6. User deactivation = remove from Clerk org + set `users.isActive = false` in DB (both in one Inngest job).
7. Password reset is handled entirely by Clerk — never build custom password flows.
8. For school onboarding: create Clerk org → invite headteacher → headteacher completes wizard. In that order.
