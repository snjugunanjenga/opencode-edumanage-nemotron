# Requirements — Sprint 005: Communications, Calendar & Procurement

**Project:** EduManage  
**Sprint:** 005  
**Status:** Approved  
**Last updated:** 2026-05-29  
**Depends on:** Sprint 004 complete

---

## Sprint Goal

Every user can communicate securely across the school, manage their unified calendar with clubs and trips, and every department can raise procurement requests through a governed approval chain. This unlocks Standard tier value and adds the remaining school operations backbone.

---

## FR-001: Direct Messaging

- 1-to-1 secure messages between any two users in the same school
- Message stored in DB; never deleted (soft-delete per user = "archived")
- Unread badge on nav sidebar; real-time updates via polling (500ms) or Pusher (if added)
- File attachments: S3 upload, max 10MB, image preview inline
- Notification: in-app bell + email (Resend) per user preference
- Message search: full-text search within user's conversations

**Data:**
```
MessageThread { id, schoolId, threadType (DIRECT|GROUP), name?, createdByUserId, createdAt }
ThreadMember { threadId, userId, schoolId, joinedAt, lastReadAt, isArchived }
Message { id, threadId, schoolId, senderId, body, attachmentS3Key?, attachmentType?, isDeleted, createdAt }
```

---

## FR-002: Group Messaging

- Pre-defined groups auto-created on school setup: "All Staff", "All Parents", "All Students"
- Class groups auto-created when classes are created: "Grade 5A Parents", "Grade 5A Students"
- Custom groups: Headteacher / Dean / Class Teacher can create
- **Safety gate:** Sending to group with >10 members shows: "You are about to send a message to N people — this cannot be recalled. Confirm?"
- Group admin can add/remove members
- Headteacher can broadcast to all users (school-wide announcement)

---

## FR-003: Notification Centre

- Bell icon in navbar — badge count of unread
- Notification types: MESSAGE | EXAM_RESULT | FEE_REMINDER | ASSIGNMENT_DUE | QUIZ_SCHEDULED | APPROVAL_REQUEST | CALENDAR_EVENT | SYSTEM_ALERT | LEAVE_APPROVED
- Per-user preferences: which types trigger email (toggle in settings)
- Mark as read (individual + "mark all read")
- Notification history: last 90 days

**Data:**
```
Notification { id, schoolId, userId, type, title, body, metadata Json, isRead, readAt?, createdAt }
NotificationPreference { userId, schoolId, type, emailEnabled, inAppEnabled }
```

---

## FR-004: Clubs & Groups

- Student joins/leaves clubs (some clubs require patron approval)
- Teacher assigned as patron (one or more clubs)
- Club has: name, description, type (academic/sports/arts/other), members, patron, schedule
- Club events: patron creates → appears on all club members' calendars
- Club progress log: patron records activity notes + achievements per meeting
- Club budget request → Accountant → Headteacher approval chain
- Trip planning: Teacher initiates → Transport Manager confirms vehicle → Headteacher approves → CalendarEvent created for all participants

**Data:**
```
Club { id, schoolId, name, description, clubType, patronUserId, requiresApproval, isActive, createdAt }
ClubMembership { clubId, userId, schoolId, joinedAt, approvedByUserId? }
ClubEvent { id, clubId, schoolId, title, startAt, endAt, location?, notes?, createdByUserId }
ClubBudgetRequest { id, clubId, schoolId, requestedByUserId, amount, purpose, status, approvedByUserId? }
```

---

## FR-005: Leave Management

- Teacher submits leave request: type (SICK | PERSONAL | OFFICIAL | MATERNITY | PATERNITY), dates, reason
- Dean receives notification + reviews: APPROVE or REJECT with note
- If approved: Dean assigns substitute teacher for affected classes (from list of available teachers that period)
- Substitute assignment: updates TimetableSlot (temporary override) + notifies substitute teacher
- Leave calendar: Dean sees all approved/pending leaves on calendar
- Headteacher can view all leave history per teacher

**Data:**
```
LeaveRequest { id, schoolId, teacherId, leaveType, startDate, endDate, reason, status (PENDING|APPROVED|REJECTED), reviewedByUserId?, reviewNote?, reviewedAt?, createdAt }
SubstituteAssignment { id, leaveRequestId, schoolId, substituteTeacherId, classId, subjectId, date }
```

---

## FR-006: Procurement System

- **Any staff role** can raise a procurement request for their department
- Multi-level approval chain (configurable per school):
  - Default: Requestor → HOD → Procurement Officer → Accountant (budget check) → Headteacher (if >threshold)
- Procurement Officer places orders with suppliers after full approval
- Delivery confirmation updates inventory
- Damaged items report → triggers reorder flag

**Approval threshold:** default Ksh 10,000 — orders above require Headteacher approval

**Data:**
```
ProcurementRequest {
  id, schoolId, requestedByUserId, department, category,
  items Json,           -- [{ name, qty, unitPrice, unit }]
  totalEstimate, currency, urgency (NORMAL|URGENT),
  status (DRAFT|PENDING_HOD|PENDING_PROCUREMENT|PENDING_ACCOUNTANT|PENDING_HEAD|APPROVED|REJECTED|ORDERED|DELIVERED),
  currentApproverId?,
  approvalChain Json,  -- [{ role, userId, status, timestamp, note }]
  createdAt
}

ProcurementOrder {
  id, requestId, schoolId, supplierName, orderDate,
  totalAmount, deliveryDate?, status (ORDERED|PARTIAL|DELIVERED|CANCELLED)
}

InventoryItem {
  id, schoolId, name, department, category,
  quantity, unit, reorderLevel, lastUpdatedAt, updatedByUserId
}

DamageReport {
  id, schoolId, itemId, reportedByUserId, quantity, description, reportedAt
}
```

**Business rules:**
- Budget check: Accountant step verifies department has sufficient budget before approval
- Headteacher step: only triggered if `totalEstimate > school.procurementApprovalThreshold`
- Rejection at any step: entire request rejected; requestor notified with reason
- Each approval step sends in-app notification to next approver

---

## Non-Functional Requirements

- Message delivery latency: <2 seconds (polling at 500ms intervals)
- Notification delivery: <5 seconds end-to-end for in-app
- Group message safety gate: never bypassable — even by Headteacher
- Procurement approval chain: full audit trail, immutable once step completed
- Leave request: cannot be approved retroactively for dates already past (unless IT override)

---

## Dashboard Tabs Delivered

**All roles:** Messages tab, Notifications bell, Calendar (enhanced with clubs/trips)  
**Dean:** Leave Queue, Substitute Scheduler, Procurement approvals  
**Procurement Officer:** Full procurement dashboard (Requests, Orders, Inventory, Damages)  
**Headteacher:** Broadcast, Procurement final approvals, Leave overview  
**Student:** Clubs tab, My Calendar (club events added)  
**Teacher:** Leave Requests tab, Clubs (patron view)
