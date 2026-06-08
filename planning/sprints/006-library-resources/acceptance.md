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
**Then** only APPROVED resources tagged to Grade 8 classes appear; IGCSE-only resources for Year 10 NOT returned; resources for Grade 9 NOT returned

**Given** no resources match the search query  
**When** search returns  
**Then** empty state shown: "No resources found for 'photosynthesis'. Ask your teacher to upload resources."

---

## AC-003: Storage Quota Enforcement

**Given** school is at 99% of their 200GB Standard quota  
**When** Teacher attempts to upload a 3GB file  
**Then** `library.resource.getUploadUrl` returns `FORBIDDEN: "Storage quota exceeded..."` — presigned URL never generated

**Given** same school at 90% quota  
**When** Inngest `recalculateStorage` job runs  
**Then** Headteacher + IT Support receive in-app notification: "Storage at 90% capacity. Current usage: 180GB of 200GB."

**Given** school upgrades from Standard to Premium  
**When** `SchoolSubscription.tier` updated to PREMIUM  
**Then** `StorageUsage.quotaBytes` updated to 1TB; uploads immediately unblocked

---

## AC-004: Parent Access Control (CRITICAL)

**Given** Parent views their child's library page  
**When** `library.student.list` returns  
**Then** response contains NO `s3Key` fields; no presigned download URLs generated for parent role

**Given** Parent attempts to call `library.resource.getDownloadUrl` for their child's personal file  
**When** tRPC called  
**Then** returns `FORBIDDEN`

**Given** Parent suggests resource "https://khanacademy.org/algebra" for child  
**When** suggestion saved  
**Then** child sees "Suggestion from Parent" badge; suggestion shows title + URL + parent's note

---

## AC-005: Share Link (Same-School Only)

**Given** Student creates a share link for their Maths notes  
**When** classmate (same school, logged in) accesses the link  
**Then** classmate can view the file; `StudentLibraryActivity` row created

**Given** user from a different school accesses the share link  
**When** `library.student.accessByToken` called  
**Then** returns `FORBIDDEN`

---

## Definition of Done

All AC-001 through AC-005 pass.  
Parent access control tested — no s3Keys or presigned URLs in parent response (CRITICAL).  
Storage quota enforcement tested end-to-end with seeded data.  
`recalculateStorage` Inngest job tested with `inngest-cli dev`.  
All new tables RLS-protected.  
`CONTEXT.md` + `STATE.md` updated.
