# Blueprint — Sprint 006: Library & Resources

**Project:** EduManage  
**Sprint:** 006  
**Last updated:** 2026-05-29

---

## New Prisma Models

```prisma
model LibraryResource {
  id               String         @id @default(uuid())
  schoolId         String
  uploadedByUserId String
  title            String
  description      String?
  resourceType     ResourceType   // NOTES | PAST_PAPER | REVISION | REFERENCE | VIDEO | OTHER
  fileS3Key        String?
  externalUrl      String?        // YouTube or external link
  mimeType         String?
  fileSizeBytes    BigInt?
  tags             Json           // String[]
  subjectId        String?
  classIds         Json           // String[] — which classes can see this
  gradeLevel       String?
  curriculumType   CurriculumType?
  syllabusTopicId  String?
  status           ResourceStatus @default(PENDING_REVIEW)
  reviewedByUserId String?
  reviewedAt       DateTime?
  rejectionReason  String?
  downloadCount    Int            @default(0)
  viewCount        Int            @default(0)
  createdAt        DateTime       @default(now())
  @@index([schoolId, status])
  @@index([schoolId, subjectId])
}

model StudentLibraryItem {
  id            String       @id @default(uuid())
  schoolId      String
  studentId     String
  title         String
  resourceType  ResourceType
  fileS3Key     String?
  externalUrl   String?
  subjectId     String?
  folderName    String?
  fileSizeBytes BigInt?
  isShared      Boolean      @default(false)
  sharedToken   String?      @unique
  createdAt     DateTime     @default(now())
  activity      StudentLibraryActivity[]
  @@index([schoolId, studentId])
}

model StudentLibraryActivity {
  id         String               @id @default(uuid())
  schoolId   String
  studentId  String
  resourceId String?              // LibraryResource
  itemId     String?              // StudentLibraryItem
  action     LibraryAction
  createdAt  DateTime             @default(now())
  item       StudentLibraryItem?  @relation(fields: [itemId], references: [id])
  @@index([schoolId, studentId, createdAt])
}

model LibraryResourceSuggestion {
  id            String   @id @default(uuid())
  schoolId      String
  parentUserId  String
  studentId     String
  title         String
  url           String
  note          String?
  isViewed      Boolean  @default(false)
  createdAt     DateTime @default(now())
  @@index([schoolId, studentId])
}

model StorageUsage {
  id               String   @id @default(uuid())
  schoolId         String   @unique
  usedBytes        BigInt   @default(0)
  quotaBytes       BigInt   // set from subscription tier
  lastCalculatedAt DateTime @default(now())
}

enum ResourceType   { NOTES PAST_PAPER REVISION REFERENCE VIDEO OTHER }
enum ResourceStatus { PENDING_REVIEW APPROVED REJECTED }
enum LibraryAction  { VIEW DOWNLOAD UPLOAD SHARE }
```

---

## tRPC Router: `library.*`

```typescript
// School library
library.resource.getUploadUrl({ filename, mimeType, fileSizeBytes })
  // 1. Check StorageUsage.usedBytes + fileSizeBytes <= quotaBytes — THROW if over
  // 2. Generate presigned POST URL
  // 3. Returns { uploadUrl, fields, resourceId (pre-created with PENDING_REVIEW) }
  // Roles: TEACHER | LIBRARIAN

library.resource.create({ title, description, resourceType, s3Key?, externalUrl?, subjectId, classIds, tags, syllabusTopicId? })
  // Called AFTER S3 upload completes to finalise the resource record
  // Roles: TEACHER (→ PENDING_REVIEW) | LIBRARIAN (→ APPROVED)

library.resource.review({ resourceId, approved, rejectionReason? })
  // Roles: LIBRARIAN only
  // Sets status = APPROVED or REJECTED; fires notification to uploader

library.resource.search({ query, subjectId?, classId?, resourceType?, cursor? })
  // Only returns APPROVED resources scoped to caller's classes
  // Roles: TEACHER | STUDENT | LIBRARIAN | DEAN | HEADTEACHER

library.resource.getDownloadUrl({ resourceId })
  // Increments viewCount via Inngest (async)
  // Returns presigned GET URL (60 min expiry)
  // Roles: TEACHER | STUDENT (own classes) | LIBRARIAN | DEAN

library.resource.list({ status?, subjectId?, uploadedByUserId?, cursor? })
  // LIBRARIAN: sees all including PENDING_REVIEW
  // TEACHER: sees own uploads + APPROVED resources for their subjects
  // Roles: LIBRARIAN | TEACHER | DEAN

// Student personal library
library.student.getUploadUrl({ filename, mimeType, fileSizeBytes })
  // Same quota check as above
  // Roles: STUDENT only

library.student.saveItem({ title, resourceType, s3Key?, externalUrl?, subjectId?, folderName? })
  // Roles: STUDENT only

library.student.list({ studentId? })
  // STUDENT: own items only
  // PARENT: own child's items (metadata only — no download URLs)
  // LIBRARIAN: any student
  // Roles: STUDENT | PARENT | LIBRARIAN

library.student.createShareLink({ itemId })
  // Generates sharedToken; returns shareable URL
  // Roles: STUDENT (own items only)

library.student.accessByToken({ token })
  // Validates token, checks user is in same school
  // Returns item metadata + presigned URL
  // Roles: STUDENT (same school)

// Parent features
library.parent.getChildActivity({ studentId, from?, to? })
  // Returns StudentLibraryActivity rows — NO download URLs
  // Roles: PARENT (own children only, validated)

library.parent.suggestResource({ studentId, title, url, note? })
  // Creates LibraryResourceSuggestion
  // Roles: PARENT (own children only)

library.student.getSuggestions()
  // Returns parent suggestions for current student
  // Roles: STUDENT (own)

// Storage
library.storage.getUsage()        // HEADTEACHER | LIBRARIAN | IT_SUPPORT
library.storage.getBreakdown()    // IT_SUPPORT only — per-user breakdown
```

---

## Inngest Jobs

```typescript
// inngest/functions/recalculate-storage.ts
// Trigger: Cron 02:00 daily
// Action:
//   1. List all S3 objects under schools/{schoolId}/ prefix
//   2. Sum sizes
//   3. Update StorageUsage.usedBytes
//   4. If > 90%: notify HT + IT Support
//   5. If > 100%: create SYSTEM_ALERT notification to block uploads (checked at upload time)

// inngest/functions/library-resource-approved.ts
// Trigger: library/resource.approved
// Action: Notify the uploader (Teacher) that their resource is approved and live

// inngest/functions/increment-download-count.ts
// Trigger: library/resource.downloaded
// Action: Increment LibraryResource.downloadCount (async — doesn't block download)
```

---

## Presigned Upload URL with Quota Check

```typescript
// server/routers/library.ts (key section)
async function checkAndGenerateUploadUrl(schoolId: string, fileSizeBytes: number, filename: string) {
  const usage = await prisma.storageUsage.findUnique({ where: { schoolId } })
  if (!usage) throw new TRPCError({ code: 'INTERNAL_SERVER_ERROR', message: 'Storage not initialised' })
  
  const projectedUsage = BigInt(usage.usedBytes) + BigInt(fileSizeBytes)
  if (projectedUsage > usage.quotaBytes) {
    const usedMB = Number(usage.usedBytes) / 1_000_000
    const quotaMB = Number(usage.quotaBytes) / 1_000_000
    throw new TRPCError({
      code: 'FORBIDDEN',
      message: `Storage quota exceeded. Used: ${usedMB.toFixed(0)}MB of ${quotaMB.toFixed(0)}MB. Please contact your school administrator to upgrade.`
    })
  }
  
  // Generate presigned POST with content-length condition
  const s3Key = `schools/${schoolId}/library/resources/${uuid()}/${filename}`
  const { url, fields } = await createPresignedPost(s3Client, {
    Bucket: process.env.AWS_S3_BUCKET!,
    Key: s3Key,
    Conditions: [
      ['content-length-range', 0, 50 * 1024 * 1024], // max 50MB
      ['starts-with', '$Content-Type', ''],
    ],
    Expires: 300, // 5 min to complete upload
  })
  
  return { uploadUrl: url, fields, s3Key }
}
```

---

## Parent Access Control (Critical)

```typescript
// server/routers/library.ts
// library.student.list — PARENT role path
if (ctx.role === 'PARENT') {
  // Validate this student is actually their child
  const student = await ctx.prisma.student.findFirst({
    where: {
      schoolId: ctx.schoolId,
      userId: studentId,
      parentIds: { has: ctx.userId } // parentIds is String[] on Student model
    }
  })
  if (!student) throw new TRPCError({ code: 'FORBIDDEN', message: 'Not your child' })
  
  // Return metadata ONLY — never include s3Key or generate download URLs for parent
  return items.map(item => ({
    id: item.id, title: item.title, resourceType: item.resourceType,
    subjectId: item.subjectId, folderName: item.folderName, createdAt: item.createdAt
    // s3Key INTENTIONALLY OMITTED
  }))
}
```

---

## Folder Structure (Sprint 006 additions)

```
app/(school)/[schoolSlug]/
└── library/
    ├── page.tsx           ← Role-aware: Librarian sees review queue; Teacher/Student sees search
    ├── upload/page.tsx    ← Teacher + Librarian upload form
    ├── review/page.tsx    ← Librarian: pending review queue
    ├── student/page.tsx   ← Student: personal library + parent suggestions
    └── admin/page.tsx     ← IT Support + Librarian: storage usage dashboard

components/library/
├── ResourceCard.tsx        ← search result card with type icon + download button
├── UploadForm.tsx          ← 'use client' — file picker + tags + subject selector
├── ReviewQueue.tsx         ← 'use client' — Librarian approve/reject list
├── StorageMeter.tsx        ← 'use client' — usage bar with tier info
└── PersonalLibrary.tsx     ← 'use client' — student folder view
```
