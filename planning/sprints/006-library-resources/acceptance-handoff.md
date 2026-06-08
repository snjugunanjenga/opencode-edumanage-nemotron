# Acceptance Criteria — Sprint 006: Library & Resources

**Last updated:** 2026-05-29

---

## AC-001: Teacher Upload → Librarian Review

**Given** Teacher uploads "Grade 8 Maths Revision Notes.pdf" (2MB)  
**When** upload completes and `library.resource.create` called  
**Then** `LibraryResource.status = PENDING_REVIEW`; Librarian sees it in review queue

**Given** Librarian approves the resource  
**When** approved  
**Then** `status = APPROVED`; Teacher receives in-app notification "Your resource has been approved"; resource now appears in student search results for tagged classes

**Given** Librarian rejects with reason "Incorrect curriculum — this is IGCSE not CBC"  
**When** rejected  
**Then** Teacher receives notification with rejection reason; resource NOT visible to students

**Given** Librarian uploads a resource directly  
**When** created  
**Then** `status = APPROVED` immediately (no review step); resource live instantly

---

## AC-002: Student Search Scoping

**Given** Student Achieng is enrolled in Grade 8A (CBC)  
**When** she searches "photosynthesis"  
**Then** only APPROVED resources tagged to Grade 8 classes appear; IGCSE-only resources for Year 10 NOT returned; resources for Grade 9 (different class) NOT returned

**Given** no resources match the search query  
**When** search returns  
**Then** empty state shown: "No resources found for 'photosynthesis'. Ask your teacher to upload resources."

---

## AC-003: Storage Quota Enforcement

**Given** school is at 99% of their 200GB Standard quota  
**When** Teacher attempts to upload a 3GB file  
**Then** `library.resource.getUploadUrl` returns `FORBIDDEN: "Storage quota exceeded..."` — presigned URL never generated; upload never starts

**Given** same school at 90% quota  
**When** Inngest `recalculateStorage` job runs  
**Then** Headteacher + IT Support receive in-app notification: "Storage at 90% capacity. Current usage: 180GB of 200GB."

**Given** school upgrades from Standard to Premium  
**When** `SchoolSubscription.tier` updated to PREMIUM  
**Then** `StorageUsage.quotaBytes` updated to 1TB (1,000,000,000,000 bytes); uploads immediately unblocked

---

## AC-004: Parent Access Control

**Given** Parent views their child's library page  
**When** `library.student.list` returns  
**Then** response contains NO `s3Key` fields; no presigned download URLs generated for parent role

**Given** Parent attempts to call `library.resource.getDownloadUrl` for their child's personal file  
**When** tRPC called  
**Then** returns `FORBIDDEN` (parent role not in allowedRoles for this procedure)

**Given** Parent suggests resource "https://khanacademy.org/algebra" for child  
**When** suggestion saved  
**Then** child sees "Suggestion from Parent" badge on their library; suggestion shows title + URL + parent's note

---

## AC-005: Share Link

**Given** Student creates a share link for their Maths notes  
**When** classmate accesses the link (logged in, same school)  
**Then** classmate can view the file; `StudentLibraryActivity` row created for the classmate

**Given** someone from a different school accesses the share link  
**When** `library.student.accessByToken` called  
**Then** returns `FORBIDDEN` (school check fails)

---

## Definition of Done

All AC-001 through AC-005 pass.  
Parent access control tested: no s3Keys or download URLs in parent response — CRITICAL.  
Storage quota enforcement tested end-to-end with seeded storage data.  
Librarian review queue working for 3+ test resources.  
`recalculateStorage` Inngest job tested with `inngest-cli dev`.  
All new tables RLS-protected.  
`CONTEXT.md` + `STATE.md` updated.

---

# Builder Handoff — Sprint 006

Paste into Claude Code. Sprint 005 must be passing.

## Pre-read (mandatory):
1. `AGENTS.md` + `CONTEXT.md`
2. `planning/DECISIONS.md` (DEC-010 storage, DEC-013 S3)
3. `planning/sprints/006-library-resources/requirements.md`
4. `planning/sprints/006-library-resources/blueprint.md`
5. `planning/sprints/006-library-resources/acceptance.md`

## Summarise before coding. Wait for approval.

## Critical rules:
1. **Quota check is synchronous** — check BEFORE generating presigned POST URL, not after
2. **Parent s3Key ban** — strip `fileS3Key` from ALL parent-role responses in `library.student.list` — tested in AC-004
3. **Download URL generation** — NEVER for PARENT role on student personal files — tRPC role guard
4. **Presigned POST policy** — include `content-length-range` condition: `['content-length-range', 0, 52428800]` (50MB max)
5. **Storage recalculation** — use S3 `ListObjectsV2` with prefix `schools/{schoolId}/` to sum sizes — do not trust DB fileSizeBytes
6. **Share links** — require valid session + same school check — NOT publicly accessible without login

## Implementation sequence:
1. Prisma migration: `sprint_006_library`
2. RLS for all 5 new tables
3. Initialize `StorageUsage` rows for all existing schools (one-time migration seed)
4. `server/routers/library.ts` — all procedures
5. Inngest jobs: recalculateStorage, libraryResourceApproved, incrementDownloadCount
6. Library pages + components
7. Tests: quota enforcement, parent access control, search scoping

## Definition of done: all AC-001 through AC-005 pass. CONTEXT.md + STATE.md updated.
