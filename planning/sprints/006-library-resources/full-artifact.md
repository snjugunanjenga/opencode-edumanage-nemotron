# Sprint 006: Library & Resources — Full Artifact

**Project:** EduManage  
**Sprint:** 006  
**Status:** Approved  
**Last updated:** 2026-05-29  
**Depends on:** Sprint 005 complete

---

# Requirements

## Sprint Goal
Students get a structured digital library per subject; teachers manage curriculum resources; the school librarian governs the system; parents monitor their child's reading activity. Storage quotas enforced by subscription tier (DEC-010: Basic 50GB / Standard 200GB / Premium 1TB).

---

## FR-001: School Library (Librarian-managed)

- Resources: `.docx`, `.pdf`, `.pptx`, `.mp4` (YouTube embed link), `.jpg/.png`
- Tags: subject, class, curriculum type (CBC/IGCSE), grade level, resource type (NOTES | PAST_PAPER | REVISION | REFERENCE | VIDEO | OTHER)
- Teacher uploads resource → Librarian review queue → approved/rejected
- Librarian can upload directly (bypasses queue)
- Resources linked to syllabus topics (optional)
- Resource search: full-text on title + description + tags

**Data:**
```
LibraryResource {
  id, schoolId, uploadedByUserId, title, description, resourceType,
  fileS3Key?, externalUrl?, mimeType?, fileSizeBytes?,
  tags Json, subjectId?, classIds Json, gradeLevel?,
  curriculumType?, syllabusTopicId?,
  status (PENDING_REVIEW | APPROVED | REJECTED),
  reviewedByUserId?, reviewedAt?, rejectionReason?,
  downloadCount, viewCount, createdAt
}
```

---

## FR-002: Teacher Subject Resources

- Teacher uploads to their assigned subjects → visible to their classes only
- Resource types same as above
- Resources appear on Student's subject page under "Teacher Resources"
- Teacher can link resource to a syllabus topic (shows progress alignment)

---

## FR-003: Student Personal Library

- Student saves/uploads: own notes, bookmarked links, downloaded resources
- Per-school storage tracked (DEC-010); IT Support + Librarian see usage dashboard
- Student can organise into folders (by subject)
- Search within own library
- Resources can be shared with classmates (generates shareable link)

**Data:**
```
StudentLibraryItem {
  id, schoolId, studentId, title, resourceType,
  fileS3Key?, externalUrl?, subjectId?,
  folderName?, isShared, sharedLink?,
  fileSizeBytes?, createdAt
}

StorageUsage {
  id, schoolId, usedBytes, quotaBytes, lastCalculatedAt
}
```

---

## FR-004: Parent Library Monitoring

- Parent views child's library activity: what was accessed, when, upload history
- Parent can suggest a resource URL to their child (appears as recommendation in child's library)
- Parent cannot view file contents — only metadata (title, date accessed)

**Data:**
```
StudentLibraryActivity { id, schoolId, studentId, resourceId?, itemId?, action (VIEW|DOWNLOAD|UPLOAD|SHARE), createdAt }
LibraryResourceSuggestion { id, schoolId, parentUserId, studentId, title, url, note?, createdAt }
```

---

## FR-005: Storage Quota Management

- Usage tracked in `StorageUsage` per school
- Inngest job recalculates usage daily (sum of all S3 object sizes for school prefix)
- When usage > 90% of quota: Headteacher + IT Support notified
- When usage > 100%: new uploads blocked; error shown to user
- IT Support can view per-user upload breakdown

---

## FR-006: Syllabus Integration

- Syllabus topics (from Sprint 003) linked to library resources
- Teacher's "Taught" progress visible alongside linked resources
- Student sees: Topic → Resources → Progress status

---

# Blueprint

## New Prisma Models
All models from FR-001 through FR-005 above. RLS on all tables.

## tRPC Router: `library.*`
```typescript
library.resource.upload({ title, description, resourceType, subjectId, classIds, tags, s3Key?, externalUrl? })   // TEACHER | LIBRARIAN
library.resource.approve({ resourceId, approve, rejectionReason? })   // LIBRARIAN
library.resource.search({ query, subjectId?, classId?, type? })       // TEACHER | STUDENT | PARENT
library.resource.download({ resourceId })                              // returns presigned S3 URL + increments downloadCount
library.resource.list({ subjectId?, classId?, status? })              // LIBRARIAN: all; TEACHER/STUDENT: APPROVED only

library.student.save({ title, s3Key?, externalUrl?, subjectId? })     // STUDENT — own items
library.student.list({ studentId? })                                  // STUDENT (own) | PARENT (own child) | LIBRARIAN
library.student.suggest({ studentId, title, url, note })              // PARENT → creates LibraryResourceSuggestion

library.storage.getUsage()                                            // IT_SUPPORT | LIBRARIAN | HEADTEACHER
library.storage.getBreakdown()                                        // IT_SUPPORT only
```

## Inngest Jobs
| Job | Trigger | Action |
|---|---|---|
| `recalculateStorageUsage` | Daily cron 02:00 | Sum S3 object sizes for school prefix → update StorageUsage |
| `storageQuotaAlert` | `storage/usage.updated` | If >90% quota → notify; if >100% → block uploads |

## S3 Structure
```
edumanage-prod-assets/
  schools/{schoolId}/
    library/resources/{resourceId}/{filename}
    library/student/{studentId}/{filename}
    exam-papers/{examId}/{filename}           ← from Sprint 003
    assignments/{assignmentId}/{filename}     ← from Sprint 003
    lesson-plans/{lessonPlanId}/{filename}
    financial-reports/{reportId}/{filename}   ← from Sprint 004
    payslips/{salaryRunItemId}/{filename}      ← from Sprint 004
```

---

# Acceptance Criteria

## AC-001: Teacher Upload → Librarian Review
- Teacher uploads resource → `status = PENDING_REVIEW`; Librarian sees it in review queue
- Librarian approves → `status = APPROVED`; resource appears in student search results
- Librarian rejects with reason → Teacher receives notification with rejection reason

## AC-002: Student Search
- Student searches "Photosynthesis" → returns all APPROVED resources tagged with matching subject/topic
- Results scoped to student's enrolled classes only — no cross-class resources

## AC-003: Storage Quota
- School at 95% quota → Headteacher + IT Support receive in-app notification
- School at 100% quota → Teacher upload attempt returns: "Storage quota exceeded. Contact your school administrator."
- After upgrading subscription tier → quota increases immediately

## AC-004: Parent Monitoring
- Parent views child's library activity → sees list of resources accessed in last 30 days
- Parent suggests a URL → student sees it in "Suggestions from Parent" section of their library
- Parent CANNOT view file content — only metadata confirmed by checking no presigned URL is generated for parent role on student personal files

## AC-005: Student Personal Library
- Student uploads a notes file → stored under their S3 prefix; appears in their library
- Student creates a shareable link → classmate can access via link (no login required? — no: shareable link requires valid session)

---

# Builder Handoff — Sprint 006

Paste into Claude Code. Sprint 005 must be passing.

**Pre-read:** all Sprint 006 planning files + `planning/DECISIONS.md` (DEC-010 storage quotas, DEC-013 S3 region).

**Summarise before coding.** Wait for approval.

**Critical rules:**
- Upload to S3: ALWAYS check `StorageUsage.usedBytes + newFileSize <= StorageUsage.quotaBytes` BEFORE generating presigned upload URL — reject early
- Student personal files: presigned URLs NEVER generated for PARENT role
- Library resources: APPROVED status required before any student/parent can access
- File size tracking: update `StorageUsage.usedBytes` in the Inngest job after every upload confirms — never trust client-provided file size
- Download count: increment via Inngest background job, not in the GET handler

**Install:**
```bash
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner  # already installed Sprint 003
```

**Definition of done:** AC-001 through AC-005 pass. Storage quota enforcement tested end-to-end. Parent file access test confirms no presigned URLs generated. CONTEXT.md + STATE.md updated.
