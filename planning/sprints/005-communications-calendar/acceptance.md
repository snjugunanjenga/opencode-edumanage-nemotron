# Acceptance Criteria — Sprint 005: Communications, Calendar & Procurement

**Last updated:** 2026-05-29

---

## AC-001: Direct Messaging

**Given** Teacher Kamau sends a direct message to Teacher Wanjiku  
**When** the message is sent  
**Then** Teacher Wanjiku sees unread badge increment by 1; message appears in thread within 2 seconds (next poll); `Notification` row created for Teacher Wanjiku with `type = MESSAGE`

**Given** Teacher Kamau sends a file attachment (PDF, 5MB)  
**When** upload completes and message sent  
**Then** inline preview shown in thread; attachment accessible via presigned URL (15-min expiry)

**Given** Teacher Kamau archives the thread  
**When** archived  
**Then** thread removed from Teacher Kamau's list (`isArchived = true`); Teacher Wanjiku still sees thread normally

**Given** Teacher Kamau tries to message a student from a different school (cross-tenant)  
**When** `messaging.createDirect` called  
**Then** tRPC throws `BAD_REQUEST: "Recipient not found in this school"`

---

## AC-002: Group Messaging Safety Gate

**Given** Headteacher sends to "All Staff" thread (45 members)  
**When** they click Send  
**Then** confirmation dialog appears: "You are about to send a message to 45 people. This cannot be recalled."

**Given** Headteacher clicks Cancel  
**When** dialog dismissed  
**Then** no message sent; message text preserved in composer input

**Given** Headteacher clicks "Send to 45 people"  
**When** confirmed  
**Then** message delivered to all 45 thread members; 45 `Notification` rows created

**Given** Class Teacher creates group with 8 members  
**When** they send a message  
**Then** NO confirmation dialog appears (threshold is >10)

---

## AC-003: Notifications

**Given** exam results released for Class 7A (30 students)  
**When** `notifications/send` Inngest job fires  
**Then** 30 `Notification` rows exist with `type = EXAM_RESULT`; students see bell badge update

**Given** Parent has `emailEnabled = false` for `EXAM_RESULT` type  
**When** result notification triggered  
**Then** in-app notification created but NO email sent via Resend; verified by checking Resend send log

**Given** user clicks "Mark all read"  
**When** `notifications.markRead()` called without `notificationId`  
**Then** all unread notifications for that user set to `isRead = true, readAt = now()`; badge count resets to 0

---

## AC-004: Clubs

**Given** Dean creates "Chess Club" (requiresApproval = true)  
**When** Student Achieng tries to join  
**Then** `ClubMembership.status = PENDING`; club patron notified

**Given** Patron approves Achieng's membership  
**When** approved  
**Then** `ClubMembership.status = ACTIVE`; Achieng notified; Achieng now sees Chess Club events on their calendar

**Given** Patron creates a club event "Chess Tournament" next Saturday  
**When** event saved  
**Then** `CalendarEvent` created with `eventType = CLUB`; appears on calendar of all ACTIVE club members

**Given** Student Achieng joins an open club (requiresApproval = false)  
**When** join submitted  
**Then** `ClubMembership.status = ACTIVE` immediately (no patron approval required)

---

## AC-005: Leave Management

**Given** Teacher Kamau requests leave for Mon–Wed next week (sick leave)  
**When** submitted  
**Then** `LeaveRequest.status = PENDING`; Dean sees new entry in Leave Queue

**Given** Dean approves with substitute Teacher Wanjiku for Kamau's Tuesday Maths Class 7A  
**When** approved  
**Then** `LeaveRequest.status = APPROVED`; `SubstituteAssignment` created; Teacher Wanjiku's calendar shows the substitution; Teacher Wanjiku receives in-app notification

**Given** Teacher tries to submit leave request with `startDate` = yesterday  
**When** `leave.request` tRPC called  
**Then** returns `BAD_REQUEST: "Leave start date cannot be in the past"`

**Given** Dean rejects leave request with note "Exam week — no leave permitted"  
**When** rejected  
**Then** `LeaveRequest.status = REJECTED`; Teacher Kamau receives notification with rejection note

---

## AC-006: Procurement

**Given** School Nurse raises a procurement request for medical supplies (Ksh 3,500)  
**When** submitted  
**Then** `status = PENDING_HOD`; relevant HOD notified (SCHOOL_NURSE HOD = HEADTEACHER directly, as Nurse reports to HT)

**Given** Procurement Officer approves at their step  
**When** approved  
**Then** `status = PENDING_ACCOUNTANT`; Accountant notified; `approvalChain` JSON has 2 entries

**Given** total estimate is Ksh 15,000 (above Ksh 10,000 threshold) and Accountant approves  
**When** Accountant approves  
**Then** `status = PENDING_HEAD` (escalates to Headteacher); Headteacher notified

**Given** any approver rejects with note "Insufficient budget"  
**When** rejected  
**Then** `status = REJECTED`; ALL prior approvers and requestor notified with rejection note; further approval actions on this request are blocked

**Given** Procurement Officer confirms delivery  
**When** `procurement.confirmDelivery` called  
**Then** `ProcurementOrder.status = DELIVERED`; `InventoryItem.quantity` updated for each delivered item; requestor notified "Your request has been delivered"

---

## Regression Check

- Sprint 002: Auth, RLS, onboarding — still passing
- Sprint 003: Academic core, calendar events from timetable — still populating
- Sprint 004: Mpesa callbacks still idempotent; PAYE calculations unchanged

---

## Definition of Done

All AC-001 through AC-006 pass.  
Group message safety gate: verified cannot be bypassed by any role.  
Procurement approval chain: full `approvalChain` JSON verified in integration test.  
Messaging RLS: verified user cannot read messages from a thread they are not a member of.  
All new tables have RLS policies.  
Inngest jobs all tested with `inngest-cli dev`.  
`CONTEXT.md` Sprint History updated. `STATE.md` updated.
