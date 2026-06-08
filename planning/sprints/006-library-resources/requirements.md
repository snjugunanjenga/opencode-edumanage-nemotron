# Requirements — Sprint 006: Library & Resources

**Project:** EduManage  
**Sprint:** 006  
**Status:** Approved  
**Last updated:** 2026-05-29  
**Depends on:** Sprint 005 complete

---

## Sprint Goal

Students get a structured digital library per subject; teachers manage curriculum resources; the school librarian governs the system with a review queue; parents monitor their child's library activity. Storage quotas enforced per subscription tier (DEC-010).

---

## FR-001: School Library (Librarian-managed)

**Roles:** Teacher uploads for review; Librarian approves/rejects; all users access approved resources

- Supported file types: `.pdf`, `.docx`, `.pptx`, `.jpg`, `.png`, `.mp4` (YouTube embed link as alternative)
- Tags: subject, class, curriculum type (CBC/IGCSE), grade level, resource type
- **Upload flow:** Teacher uploads → `status = PENDING_REVIEW` → Librarian reviews → APPROVED or REJECTED with reason
- Librarian can upload directly (bypasses review — auto-APPROVED)
- Resources optionally linked to syllabus topics
- Full-text search: title + description + tags
- Download count + view count tracked per resource
- Resources scoped to enrolled class — students only see resources tagged to their classes

---

## FR-002: Teacher Subject Resources

- Teacher uploads to their assigned subjects — visible to their assigned classes only
- Same resource types as school library
- Resources appear on student's subject page under "Teacher Resources" tab
- Teacher can link resource to a specific syllabus topic (shows alignment)
- Dean can view all teacher-uploaded resources for any subject

---

## FR-003: Student Personal Library

- Student uploads/saves own notes (`.docx`, `.pdf`) and bookmarks links
- Organised into subject folders
- Per-school storage tracked (DEC-010 tier quotas)
- Search within own personal library
- Share option: generates a link accessible by classmates (requires valid session — not public)
- Parent can see activity metadata but NOT file contents

---

## FR-004: Parent Library Monitoring

- Parent views child's library activity: resources accessed, when, uploads
- Parent suggests resource to child: title + URL + optional note → appears as recommendation in child's library
- Parent CANNOT access presigned download URLs for student personal files — metadata only

---

## FR-005: Storage Quota Management

**Tier quotas (DEC-010):** Basic 50GB / Standard 200GB / Premium 1TB per school

- Quota checked BEFORE generating presigned upload URL — rejected if would exceed quota
- `StorageUsage` table updated by Inngest job (daily S3 size recalculation)
- 90% threshold: Headteacher + IT Support notified
- 100% threshold: all uploads blocked with clear error message
- IT Support + Librarian: storage usage dashboard showing per-user breakdown

---

## FR-006: Syllabus Integration

- Library resources can be tagged to syllabus topics (from Sprint 003 `SyllabusTopic`)
- Teacher marks topic as "taught" → resources linked to that topic highlighted as "covered"
- Student views: Topic list → linked resources per topic → teacher's progress status

---

## Resource Types

| Type | Description |
|---|---|
| NOTES | Lecture notes, study guides |
| PAST_PAPER | Previous exam papers |
| REVISION | Revision materials |
| REFERENCE | Reference documents, textbooks |
| VIDEO | YouTube embed link or uploaded video |
| OTHER | Uncategorised |

---

## Storage Architecture

```
S3 Bucket: edumanage-prod-assets/
  schools/{schoolId}/
    library/resources/{resourceId}/{originalFilename}
    library/student/{studentId}/{filename}
```

All school files use the school's S3 prefix. Quota = sum of all object sizes under `schools/{schoolId}/`.

---

## Non-Functional Requirements

- Library search results: <500ms (indexed on `schoolId + status + subjectId`)
- Presigned URL expiry: 60 minutes (longer than other features — reading/downloading takes time)
- Storage quota check: synchronous in upload URL generation — never allow over-quota uploads
- Librarian review queue: shows new submissions in reverse chronological order, oldest-first option
- Max file size: 50MB per file (enforced via S3 presigned POST policy condition)
