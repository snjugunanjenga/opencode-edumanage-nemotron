# Blueprint — Sprint 005: Communications, Calendar & Procurement

**Last updated:** 2026-05-29

---

## New Prisma Models

All models from requirements.md FR-001 through FR-006 (MessageThread, ThreadMember, Message, Notification, NotificationPreference, Club, ClubMembership, ClubEvent, ClubBudgetRequest, LeaveRequest, SubstituteAssignment, ProcurementRequest, ProcurementOrder, InventoryItem, DamageReport).

RLS policy: `school_id` on every table.

---

## tRPC Routers

### `messaging.*`
- `getThreads()` — list user's conversations
- `getMessages({ threadId, cursor })` — paginated message history
- `sendMessage({ threadId, body, attachmentS3Key? })` — creates Message + triggers notification Inngest job
- `createDirectThread({ recipientUserId })` — or return existing thread
- `createGroupThread({ name, memberUserIds })` — HEADTEACHER | DEAN | CLASS_TEACHER; validates members ≤ school
- `markRead({ threadId })` — updates ThreadMember.lastReadAt
- `archiveThread({ threadId })` — soft-delete for current user

### `notifications.*`
- `getMyNotifications({ cursor })` — paginated, newest first
- `markRead({ notificationId? })` — if null: mark all read
- `getPreferences()` + `updatePreferences({ type, emailEnabled, inAppEnabled })`

### `clubs.*`
- `create({ name, description, clubType, requiresApproval })` — HEADTEACHER | DEAN
- `join({ clubId })` — STUDENT (if !requiresApproval: immediate; else pending patron approval)
- `approveMember({ clubId, userId })` — club.patronUserId
- `createEvent({ clubId, title, startAt, endAt })` — patron → auto-creates CalendarEvent
- `logProgress({ clubId, notes })` — patron
- `requestBudget({ clubId, amount, purpose })` — patron

### `leave.*`
- `request({ leaveType, startDate, endDate, reason })` — CLASS_TEACHER | SUBJECT_TEACHER
- `review({ requestId, status, note, substituteTeacherId? })` — DEAN_ACADEMICS
- `getQueue()` — DEAN_ACADEMICS: pending requests
- `getMyHistory()` — own leave history

### `procurement.*`
- `create({ department, category, items[], urgency })` — any staff role
- `approve({ requestId, note? })` — current step's approver
- `reject({ requestId, note })` — any approver in chain
- `createOrder({ requestId, supplierName, orderDate, totalAmount })` — PROCUREMENT_OFFICER
- `confirmDelivery({ orderId, items[] })` — PROCUREMENT_OFFICER → updates inventory
- `reportDamage({ itemId, quantity, description })` — any staff
- `getInventory({ department? })` — PROCUREMENT_OFFICER + dept heads

---

## Inngest Jobs

| Job | Trigger | Action |
|---|---|---|
| `sendMessageNotification` | `messaging/message.sent` | In-app notification + conditional email per ThreadMember preferences |
| `sendNotification` | `notifications/send` | Generic — creates Notification row + sends email if preference set |
| `procurementNextApprover` | `procurement/step.completed` | Determine next approver → notify them |
| `leaveApproved` | `leave/request.approved` | Update timetable slots with substitute → notify substitute teacher |

---

## Group Message Safety Gate (Client)

```typescript
// components/messaging/GroupMessageComposer.tsx
'use client'
async function handleSend() {
  if (thread.memberCount > 10) {
    const confirmed = await showConfirmDialog({
      title: 'Send to group',
      message: `You are about to send this message to ${thread.memberCount} people. This cannot be recalled.`,
      confirmText: 'Send to all',
      variant: 'warning',
    })
    if (!confirmed) return
  }
  await sendMessage.mutateAsync({ threadId: thread.id, body })
}
```

---

## Procurement Approval State Machine

```
DRAFT → PENDING_HOD (on submit)
PENDING_HOD → PENDING_PROCUREMENT (HOD approves)
           → REJECTED (HOD rejects)
PENDING_PROCUREMENT → PENDING_ACCOUNTANT (Procurement Officer approves)
                    → REJECTED
PENDING_ACCOUNTANT → APPROVED (if totalEstimate ≤ threshold)
                   → PENDING_HEAD (if totalEstimate > threshold)
                   → REJECTED
PENDING_HEAD → APPROVED (Headteacher approves)
             → REJECTED
APPROVED → ORDERED (Procurement Officer places order)
ORDERED → DELIVERED (delivery confirmed)
```

---

## New npm Packages

```bash
npm install pusher-js pusher  # optional real-time — use polling first, add Pusher if needed
```

---

## Real-time Strategy

- **v1 (this sprint):** 500ms polling via tRPC `getThreads` + `getMyNotifications`. Simple, zero infra.
- **v2 upgrade path:** Replace with Pusher Channels. Architect has documented this migration path.

---

# Acceptance Criteria — Sprint 005

## AC-001: Direct Messaging
- User A sends message to User B → User B sees unread badge; message appears in thread within 1s (next poll)
- File attachment uploaded → inline preview shown in thread
- User archives thread → thread removed from their list; other user still sees it

## AC-002: Group Safety Gate
- Headteacher sends to "All Staff" (>10 members) → confirmation dialog appears with exact member count
- Headteacher cancels → no message sent
- Headteacher confirms → message delivered to all members

## AC-003: Notifications
- Exam results released → all affected students receive in-app notification within 5 seconds
- User with `emailEnabled = false` for EXAM_RESULT type → no email sent, but in-app notification created
- "Mark all read" → all `Notification.isRead = true` for that user

## AC-004: Clubs
- Student joins open club → `ClubMembership` created immediately
- Student joins restricted club → `status = PENDING`; patron sees approval request
- Patron creates club event → CalendarEvent created for all club members with `eventType = CLUB`

## AC-005: Leave Management
- Teacher requests leave → Dean sees it in Leave Queue
- Dean approves with substitute Teacher Wanjiku for affected periods → Teacher Wanjiku's timetable updated + notified
- Teacher requests leave for dates already past → tRPC returns BAD_REQUEST

## AC-006: Procurement
- Any staff member raises procurement request → first approver (HOD) notified
- HOD approves → Procurement Officer notified
- Procurement Officer rejects → requestor notified with rejection note; status = REJECTED
- Order delivered → inventory quantities updated; requestor notified

---

# Builder Handoff — Sprint 005

Paste into Claude Code. Sprint 004 must be passing.

**Pre-read:** AGENTS.md, CONTEXT.md, all Sprint 005 planning files.

**Summarise before coding** (files, migrations, Inngest jobs, env vars, ARCH-QUESTIONs). Wait for approval.

**Critical rules:**
- Group message gate: NEVER skip the confirmation dialog for groups >10, regardless of role
- Procurement approval chain: each step is immutable once completed — no edits to `approvalChain` JSON after step written
- Notifications: fire via Inngest `sendNotification` function — never inline in tRPC mutations
- Messaging: poll at 500ms for v1 — do NOT add WebSocket infrastructure this sprint
- Leave: Dean cannot approve retroactive leave (startDate < today) without IT override flag

**Definition of done:** AC-001 through AC-006 pass. All new tables RLS-protected. CONTEXT.md + STATE.md updated.
