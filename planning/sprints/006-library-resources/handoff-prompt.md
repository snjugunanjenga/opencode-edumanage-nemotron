# Builder Handoff — Sprint 006: Library & Resources

Paste everything below into Claude Code. Sprint 005 must be fully passing.

---

You are building Sprint 006 of EduManage: the digital library system with storage quota enforcement, librarian review workflow, student personal library, and parent monitoring.

## Pre-read (mandatory, in order):
1. `AGENTS.md` + `CONTEXT.md`
2. `planning/DECISIONS.md` (DEC-010 storage quotas, DEC-013 S3 eu-west-1)
3. `planning/RISKS.md` (RISK-005 S3 data residency)
4. `planning/sprints/006-library-resources/requirements.md`
5. `planning/sprints/006-library-resources/blueprint.md`
6. `planning/sprints/006-library-resources/acceptance.md`

## Agent routing:
- Prisma models, tRPC router, S3 presigned logic → `.claude/agents/nextjs-expert.md`
- Library search UI, upload form, resource cards, storage meter → `.claude/agents/react-frontend.md`
- Role guards, parent access control, share token validation → `.claude/agents/clerk-auth.md`
- Inngest storage recalc job, S3 ListObjects, CI → `.claude/agents/devops.md`

## Summarise before coding — output:
1. New files (full paths)
2. Modified files + what changes
3. Prisma migration name
4. Inngest job names
5. One-time data migration (StorageUsage seed)
6. Any `// ARCH-QUESTION:` items

Wait for operator approval. Then build.

## Implementation sequence:

### Phase 1: Database + Storage Init
1. `prisma migrate dev --name sprint_006_library`
2. RLS SQL: `prisma/migrations/manual/005_rls_sprint006.sql` — standard tenant isolation for all 5 new tables
3. Seed `StorageUsage` rows for all existing schools:
   ```sql
   INSERT INTO storage_usages (id, school_id, used_bytes, quota_bytes, last_calculated_at)
   SELECT gen_random_uuid(), id, 0,
     CASE subscription_tier
       WHEN 'BASIC' THEN 53687091200      -- 50GB
       WHEN 'STANDARD' THEN 214748364800  -- 200GB
       WHEN 'PREMIUM' THEN 1099511627776  -- 1TB
     END,
     now()
   FROM schools
   ON CONFLICT (school_id) DO NOTHING;
   ```

### Phase 2: tRPC Router
4. `server/routers/library.ts` — all procedures from blueprint.md
5. **CRITICAL in getUploadUrl**: synchronous quota check before presigned POST generation
6. **CRITICAL in student.list (PARENT role)**: strip all `fileS3Key` + `externalUrl` fields — return metadata only
7. Add `library` router to `server/routers/index.ts`

### Phase 3: Inngest Jobs
8. `inngest/functions/recalculate-storage.ts` — daily cron, S3 ListObjectsV2, upsert StorageUsage, alert at 90%/100%
9. `inngest/functions/library-resource-approved.ts` — notify teacher on approval
10. `inngest/functions/increment-download-count.ts` — async counter increment

### Phase 4: UI
11. `/library/page.tsx` — role-aware: Librarian sees review queue tab; Teacher sees upload + search; Student sees school library + personal
12. `components/library/UploadForm.tsx` — file picker, tags, subject selector, progress indicator
13. `components/library/ReviewQueue.tsx` — Librarian approve/reject with reason
14. `components/library/ResourceCard.tsx` — title, type icon, subject tag, download button
15. `components/library/StorageMeter.tsx` — usage bar, tier label, upgrade CTA
16. `components/library/PersonalLibrary.tsx` — student folder view, upload, share link

### Phase 5: Tests
17. Integration: quota check blocks upload when over limit
18. Integration: parent receives no s3Key in response (parse response and assert)
19. Integration: search returns only classes student is enrolled in
20. Integration: share link cross-school access returns FORBIDDEN
21. Inngest test: recalculateStorage updates StorageUsage.usedBytes correctly

## Critical rules:
1. Quota check SYNCHRONOUS — before presigned URL, not after
2. Parent s3Key ban — test explicitly in AC-004
3. Presigned POST conditions: `['content-length-range', 0, 52428800]`
4. Storage recalculation: use S3 `ListObjectsV2` — never trust DB fileSizeBytes
5. Share links: require valid session + same schoolId — never publicly accessible

## Definition of done:
All AC-001 through AC-005 pass. Parent access control explicitly tested. Storage quota enforcement tested. CONTEXT.md + STATE.md updated.
