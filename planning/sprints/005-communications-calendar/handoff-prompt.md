# Builder Handoff — Sprint 005: Communications, Calendar & Procurement

Paste everything below into Claude Code. Sprint 004 must be fully passing.

---

You are building Sprint 005 of EduManage — messaging, notifications, clubs, leave management, and the full procurement system.

## Pre-read (mandatory, in order):
1. `AGENTS.md` + `CONTEXT.md`
2. `planning/DOMAIN.md` (all 16 roles, procurement business rules)
3. `planning/DECISIONS.md` (all decisions)
4. `planning/sprints/005-communications-calendar/requirements.md`
5. `planning/sprints/005-communications-calendar/blueprint.md`
6. `planning/sprints/005-communications-calendar/acceptance.md`

## Agent routing:
- Prisma models, tRPC routers → `nextjs-expert`
- Message UI, notification bell, club pages → `react-frontend`
- Role guards, thread membership checks → `clerk-auth`
- Inngest jobs, polling hooks → `devops`

## Summarise before coding — output:
1. New files (full paths)
2. Modified files + what changes
3. Prisma migration name
4. Inngest job names
5. Any `// ARCH-QUESTION:` items

Wait for operator approval. Then build.

## Implementation sequence:

### Phase 1: Database
1. `prisma migrate dev --name sprint_005_communications_procurement`
2. RLS SQL: `prisma/migrations/manual/004_rls_sprint005.sql`
   - Standard tenant isolation on all new tables
   - Special message RLS: users can only read messages in threads they are members of
   - Special notification RLS: users can only read their own notifications

### Phase 2: Messaging tRPC router
3. `server/routers/messaging.ts` — getThreads, getMessages (cursor-based pagination), sendMessage, createDirect, createGroup, markRead, archiveThread
4. `server/routers/notifications.ts` — getMyNotifications, markRead, getPreferences, updatePreferences, getUnreadCount
5. Update `server/routers/index.ts` with new routers

### Phase 3: Procurement & Leave routers
6. `server/routers/clubs.ts` — all club procedures
7. `server/routers/leave.ts` — request, getQueue, review, getMyHistory, getSchoolOverview
8. `server/routers/procurement.ts` — full state machine (create, submit, approve, reject, createOrder, confirmDelivery, reportDamage, getInventory, updateInventory)

### Phase 4: Inngest jobs
9. `inngest/functions/messaging-notification.ts`
10. `inngest/functions/send-notification.ts` (generic — used by all modules)
11. `inngest/functions/procurement-next-approver.ts`
12. `inngest/functions/leave-approved.ts`

### Phase 5: UI
13. Messages page: `app/(school)/[schoolSlug]/communications/page.tsx`
14. Thread list (sidebar) + message view (main panel) + composer
15. Group composer with safety gate dialog (`components/messaging/GroupComposer.tsx`)
16. Notification bell with dropdown (`components/shared/NotificationBell.tsx`) — add to all dashboard layouts
17. Club pages: list, detail, join flow, event creation
18. Leave management: teacher request form, Dean review queue
19. Procurement pages: create request form, approval queue, inventory view

### Phase 6: Polling hook
20. `hooks/useMessages.ts` — 2s polling via tRPC query invalidation
21. `hooks/useUnreadCount.ts` — 10s polling for notification badge

### Phase 7: Tests
22. Integration: messaging RLS (user cannot read messages in thread they don't belong to)
23. Integration: group safety gate — confirm dialog appears when memberCount > 10
24. Integration: procurement state machine — full happy path (DRAFT → DELIVERED)
25. Integration: procurement rejection — blocked after rejection
26. E2E: send direct message → recipient sees notification

## Critical rules:
- **Group safety gate is MANDATORY** — not optional, not bypassable by any role
- **Message RLS**: implement at BOTH tRPC level (check thread membership) AND Postgres RLS level — belt and braces
- **Procurement `approvalChain`**: append-only JSON array — never mutate existing entries
- **Notifications**: ALWAYS fire via Inngest — never inline in mutations (keeps mutations fast)
- **Leave retroactive check**: `if (startDate < today) throw BAD_REQUEST` — no exceptions without IT flag
- **Procurement delivery**: `InventoryItem.quantity` update must be in same Inngest transaction as `OrderStatus = DELIVERED` — never partial
- **Polling**: use `useQuery` + `refetchInterval` NOT `useEffect + setInterval` to avoid memory leaks

## Definition of done:
All AC-001 through AC-006 pass.  
Group safety gate tested and confirmed non-bypassable.  
Messaging RLS verified (cross-thread access returns 0 rows).  
Full procurement chain tested from DRAFT to DELIVERED.  
All 4 Inngest jobs tested with `inngest-cli dev`.  
`CONTEXT.md` Sprint History updated. `STATE.md` updated.
