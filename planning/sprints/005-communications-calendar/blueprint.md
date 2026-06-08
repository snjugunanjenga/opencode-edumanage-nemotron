# Blueprint — Sprint 005: Communications, Calendar & Procurement

**Project:** EduManage  
**Sprint:** 005  
**Status:** Approved  
**Last updated:** 2026-05-29  
**Depends on:** Sprint 004 complete

---

## New Prisma Models

```prisma
// ─── Messaging ───────────────────────────────────────────────────

model MessageThread {
  id             String       @id @default(uuid())
  schoolId       String
  threadType     ThreadType   // DIRECT | GROUP
  name           String?      // group threads only
  createdByUserId String
  createdAt      DateTime     @default(now())
  members        ThreadMember[]
  messages       Message[]
  @@index([schoolId])
}

model ThreadMember {
  threadId     String
  userId       String
  schoolId     String
  joinedAt     DateTime @default(now())
  lastReadAt   DateTime?
  isArchived   Boolean  @default(false)
  thread       MessageThread @relation(fields: [threadId], references: [id])
  @@id([threadId, userId])
  @@index([schoolId, userId])
}

model Message {
  id              String        @id @default(uuid())
  threadId        String
  schoolId        String
  senderId        String
  body            String
  attachmentS3Key String?
  attachmentType  String?       // "image" | "document" | "pdf"
  isDeleted       Boolean       @default(false)
  createdAt       DateTime      @default(now())
  thread          MessageThread @relation(fields: [threadId], references: [id])
  @@index([schoolId, threadId, createdAt])
}

enum ThreadType { DIRECT GROUP }

// ─── Notifications ───────────────────────────────────────────────

model Notification {
  id        String           @id @default(uuid())
  schoolId  String
  userId    String
  type      NotificationType
  title     String
  body      String
  metadata  Json?            // { entityId, entityType, url }
  isRead    Boolean          @default(false)
  readAt    DateTime?
  createdAt DateTime         @default(now())
  @@index([schoolId, userId, isRead])
  @@index([schoolId, userId, createdAt])
}

model NotificationPreference {
  userId        String
  schoolId      String
  type          NotificationType
  emailEnabled  Boolean  @default(true)
  inAppEnabled  Boolean  @default(true)
  @@id([userId, schoolId, type])
}

enum NotificationType {
  MESSAGE EXAM_RESULT FEE_REMINDER ASSIGNMENT_DUE QUIZ_SCHEDULED
  APPROVAL_REQUEST CALENDAR_EVENT SYSTEM_ALERT LEAVE_APPROVED
  PROCUREMENT_UPDATE HEALTH_ALERT REPORT_READY
}

// ─── Clubs ───────────────────────────────────────────────────────

model Club {
  id               String          @id @default(uuid())
  schoolId         String
  name             String
  description      String?
  clubType         ClubType
  patronUserId     String
  requiresApproval Boolean         @default(false)
  isActive         Boolean         @default(true)
  createdAt        DateTime        @default(now())
  members          ClubMembership[]
  events           ClubEvent[]
  budgetRequests   ClubBudgetRequest[]
  @@index([schoolId])
}

model ClubMembership {
  clubId           String
  userId           String
  schoolId         String
  status           MemberStatus  @default(ACTIVE)  // ACTIVE | PENDING | INACTIVE
  joinedAt         DateTime      @default(now())
  approvedByUserId String?
  club             Club          @relation(fields: [clubId], references: [id])
  @@id([clubId, userId])
  @@index([schoolId, userId])
}

model ClubEvent {
  id              String   @id @default(uuid())
  clubId          String
  schoolId        String
  title           String
  startAt         DateTime
  endAt           DateTime
  location        String?
  notes           String?
  createdByUserId String
  createdAt       DateTime @default(now())
  club            Club     @relation(fields: [clubId], references: [id])
  @@index([schoolId, clubId])
}

model ClubBudgetRequest {
  id               String  @id @default(uuid())
  clubId           String
  schoolId         String
  requestedByUserId String
  amount           Decimal @db.Decimal(12,2)
  purpose          String
  status           String  @default("PENDING")  // PENDING | APPROVED | REJECTED
  approvedByUserId String?
  club             Club    @relation(fields: [clubId], references: [id])
  createdAt        DateTime @default(now())
  @@index([schoolId])
}

enum ClubType   { ACADEMIC SPORTS ARTS SCIENCE DRAMA MUSIC CHESS OTHER }
enum MemberStatus { ACTIVE PENDING INACTIVE }

// ─── Leave Management ─────────────────────────────────────────────

model LeaveRequest {
  id              String      @id @default(uuid())
  schoolId        String
  teacherId       String
  leaveType       LeaveType
  startDate       DateTime    @db.Date
  endDate         DateTime    @db.Date
  reason          String
  status          LeaveStatus @default(PENDING)
  reviewedByUserId String?
  reviewNote      String?
  reviewedAt      DateTime?
  createdAt       DateTime    @default(now())
  substitutions   SubstituteAssignment[]
  @@index([schoolId, teacherId])
  @@index([schoolId, status])
}

model SubstituteAssignment {
  id                  String       @id @default(uuid())
  leaveRequestId      String
  schoolId            String
  substituteTeacherId String
  classId             String
  subjectId           String
  date                DateTime     @db.Date
  leaveRequest        LeaveRequest @relation(fields: [leaveRequestId], references: [id])
  @@index([schoolId, substituteTeacherId])
}

enum LeaveType  { SICK PERSONAL OFFICIAL MATERNITY PATERNITY }
enum LeaveStatus { PENDING APPROVED REJECTED }

// ─── Procurement ─────────────────────────────────────────────────

model ProcurementRequest {
  id                String         @id @default(uuid())
  schoolId          String
  requestedByUserId String
  department        String
  category          String         // SUPPLIES | EQUIPMENT | REPAIR | MEDICAL | FOOD | OTHER
  items             Json           // [{ name, qty, unitPrice, unit }]
  totalEstimate     Decimal        @db.Decimal(12,2)
  currency          String         @default("KES")
  urgency           Urgency        @default(NORMAL)
  status            ProcurementStatus @default(DRAFT)
  currentApproverId String?
  approvalChain     Json           @default("[]")
  rejectionReason   String?
  createdAt         DateTime       @default(now())
  orders            ProcurementOrder[]
  @@index([schoolId, status])
  @@index([schoolId, requestedByUserId])
}

model ProcurementOrder {
  id           String            @id @default(uuid())
  requestId    String
  schoolId     String
  supplierName String
  orderDate    DateTime          @db.Date
  totalAmount  Decimal           @db.Decimal(12,2)
  deliveryDate DateTime?         @db.Date
  status       OrderStatus       @default(ORDERED)
  request      ProcurementRequest @relation(fields: [requestId], references: [id])
  @@index([schoolId])
}

model InventoryItem {
  id              String   @id @default(uuid())
  schoolId        String
  name            String
  department      String
  category        String
  quantity        Decimal  @db.Decimal(10,2)
  unit            String
  reorderLevel    Decimal  @db.Decimal(10,2)
  lastUpdatedAt   DateTime @updatedAt
  updatedByUserId String
  @@index([schoolId, department])
}

model DamageReport {
  id               String   @id @default(uuid())
  schoolId         String
  itemId           String
  reportedByUserId String
  quantity         Decimal  @db.Decimal(10,2)
  description      String
  reportedAt       DateTime @default(now())
  @@index([schoolId])
}

enum Urgency          { NORMAL URGENT }
enum ProcurementStatus {
  DRAFT PENDING_HOD PENDING_PROCUREMENT PENDING_ACCOUNTANT PENDING_HEAD
  APPROVED REJECTED ORDERED DELIVERED CANCELLED
}
enum OrderStatus      { ORDERED PARTIAL DELIVERED CANCELLED }
```

---

## tRPC Routers

### `messaging.*`
```typescript
messaging.getThreads()                              // ALL roles
messaging.getMessages({ threadId, cursor? })        // thread member only
messaging.sendMessage({ threadId, body, attachmentS3Key? }) // thread member
messaging.createDirect({ recipientUserId })         // ALL roles — returns existing if found
messaging.createGroup({ name, memberUserIds[] })    // HT | DEAN | CLASS_TEACHER
messaging.markRead({ threadId })                    // thread member
messaging.archiveThread({ threadId })               // thread member
```

### `notifications.*`
```typescript
notifications.getMyNotifications({ cursor? })       // ALL roles
notifications.markRead({ notificationId? })         // own notifications only
notifications.getPreferences()                      // ALL roles
notifications.updatePreferences({ type, emailEnabled, inAppEnabled }) // own prefs
notifications.getUnreadCount()                      // ALL roles — used for badge
```

### `clubs.*`
```typescript
clubs.create({ name, description, clubType, requiresApproval }) // HT | DEAN
clubs.list()                                         // ALL
clubs.join({ clubId })                               // STUDENT
clubs.approveJoin({ clubId, userId })                // club patron
clubs.createEvent({ clubId, title, startAt, endAt, location? }) // club patron
clubs.logProgress({ clubId, notes })                 // club patron
clubs.requestBudget({ clubId, amount, purpose })     // club patron
clubs.approveBudget({ requestId })                   // ACCOUNTANT | HT
```

### `leave.*`
```typescript
leave.request({ leaveType, startDate, endDate, reason })              // CLASS_TEACHER | SUBJECT_TEACHER
leave.getQueue()                                                       // DEAN_ACADEMICS
leave.review({ requestId, status, reviewNote, substituteTeacherId? }) // DEAN_ACADEMICS
leave.getMyHistory()                                                   // own role
leave.getSchoolOverview({ termId? })                                  // HT | DEAN
```

### `procurement.*`
```typescript
procurement.create({ department, category, items[], urgency })     // ANY staff
procurement.submit({ requestId })                                  // requestor
procurement.approve({ requestId, note? })                          // current approver in chain
procurement.reject({ requestId, note })                            // any approver
procurement.createOrder({ requestId, supplierName, totalAmount })  // PROCUREMENT_OFFICER
procurement.confirmDelivery({ orderId, deliveredItems[] })         // PROCUREMENT_OFFICER
procurement.reportDamage({ itemId, quantity, description })        // ANY staff
procurement.getInventory({ department? })                          // PROCUREMENT_OFFICER + HODs
procurement.updateInventory({ itemId, quantity })                  // PROCUREMENT_OFFICER
procurement.listRequests({ status?, department? })                 // PROCUREMENT_OFFICER | HT | ACCT
```

---

## Procurement Approval State Machine

```
DRAFT
  └── submit() → PENDING_HOD
        ├── HOD approves → PENDING_PROCUREMENT
        │     ├── ProcurementOfficer approves → PENDING_ACCOUNTANT
        │     │     ├── Accountant approves (≤threshold) → APPROVED
        │     │     ├── Accountant approves (>threshold) → PENDING_HEAD
        │     │     │     ├── Headteacher approves → APPROVED
        │     │     │     └── Headteacher rejects → REJECTED
        │     │     └── Accountant rejects → REJECTED
        │     └── ProcurementOfficer rejects → REJECTED
        └── HOD rejects → REJECTED

APPROVED
  └── createOrder() → ORDERED
        └── confirmDelivery() → DELIVERED
```

Each state transition:
1. Validates current user IS the current step's approver
2. Updates `status` + `approvalChain` JSON (appends `{ role, userId, action, timestamp, note }`)
3. Fires Inngest `procurement/step.completed` → notifies next approver

---

## Inngest Jobs (Sprint 005)

```typescript
// inngest/functions/messaging-notification.ts
// Trigger: messaging/message.sent
// Action: For each thread member (except sender): create Notification row
//         If emailEnabled for MESSAGE type: send email via Resend

// inngest/functions/send-notification.ts  
// Trigger: notifications/send (generic)
// Action: Create Notification row + conditional email

// inngest/functions/procurement-next-approver.ts
// Trigger: procurement/step.completed
// Action: Determine next approver from chain → notify them

// inngest/functions/leave-approved.ts
// Trigger: leave/request.approved
// Action: Update TimetableSlot(s) with substituteTeacherId (temp override)
//         Create CalendarEvent for substitute teacher
//         Notify substitute teacher
```

---

## Group Message Safety Gate Implementation

```typescript
// components/messaging/GroupComposer.tsx
'use client'
import { useConfirm } from '@/components/ui/confirm-dialog'

export function GroupComposer({ thread, onSend }) {
  const confirm = useConfirm()

  async function handleSend(body: string) {
    if (thread.memberCount > 10) {
      const ok = await confirm({
        title: 'Send group message',
        description: `You are about to send a message to ${thread.memberCount} people. This cannot be recalled.`,
        confirmLabel: `Send to ${thread.memberCount} people`,
        variant: 'destructive',
      })
      if (!ok) return
    }
    await sendMessage({ threadId: thread.id, body })
  }
  // ...
}
```

---

## Real-time Strategy (v1 = Polling)

```typescript
// hooks/useMessages.ts
'use client'
import { useEffect, useRef } from 'react'
import { api } from '@/lib/trpc/client'

export function useMessages(threadId: string) {
  const utils = api.useUtils()
  
  useEffect(() => {
    const interval = setInterval(() => {
      utils.messaging.getMessages.invalidate({ threadId })
    }, 2000) // 2s polling in v1
    return () => clearInterval(interval)
  }, [threadId, utils])

  return api.messaging.getMessages.useInfiniteQuery({ threadId })
}
```

Upgrade path to Pusher documented in RISKS.md RISK-009 (add when needed).

---

## RLS Additions (Sprint 005)

```sql
-- All Sprint 005 tables
ALTER TABLE message_threads ENABLE ROW LEVEL SECURITY;
ALTER TABLE thread_members ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification_preferences ENABLE ROW LEVEL SECURITY;
ALTER TABLE clubs ENABLE ROW LEVEL SECURITY;
ALTER TABLE club_memberships ENABLE ROW LEVEL SECURITY;
ALTER TABLE club_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE club_budget_requests ENABLE ROW LEVEL SECURITY;
ALTER TABLE leave_requests ENABLE ROW LEVEL SECURITY;
ALTER TABLE substitute_assignments ENABLE ROW LEVEL SECURITY;
ALTER TABLE procurement_requests ENABLE ROW LEVEL SECURITY;
ALTER TABLE procurement_orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE inventory_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE damage_reports ENABLE ROW LEVEL SECURITY;

-- Standard tenant isolation for all above
CREATE POLICY tenant_isolation ON message_threads USING (school_id = current_setting('app.current_school_id',true)::uuid);
-- ... (repeat pattern for all tables)

-- Messages: only thread members can read
CREATE POLICY message_thread_member ON messages
  USING (
    school_id = current_setting('app.current_school_id',true)::uuid
    AND EXISTS (
      SELECT 1 FROM thread_members tm
      WHERE tm.thread_id = messages.thread_id
      AND tm.user_id = current_setting('app.current_user_id',true)::uuid
      AND tm.is_archived = false
    )
  );

-- Notifications: users see only their own
CREATE POLICY own_notifications ON notifications
  USING (
    school_id = current_setting('app.current_school_id',true)::uuid
    AND user_id = current_setting('app.current_user_id',true)::uuid
  );
```

---

## New npm Packages

```bash
# No new packages needed — all functionality uses existing stack
# Pusher added only if polling proves inadequate (v2 decision)
```
