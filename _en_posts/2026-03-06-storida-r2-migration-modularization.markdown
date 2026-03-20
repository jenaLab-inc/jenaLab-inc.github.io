---
layout: post
ref: storida-r2-migration-modularization
title: "Cloudflare R2 Migration and Scheduler Modularization"
date: 2026-03-06 10:00:00
categories: infrastructure
author: jenalab
image: /assets/article_images/storida/r2-migration.svg
image2: /assets/article_images/storida/r2-migration.svg
---

Two changes shipped on the same day. R2 migration for storage costs. Scheduler modularization to make the migration manageable. They reinforced each other.

## Part 1: Supabase Storage to Cloudflare R2

### Why Migrate

Supabase Storage worked for early development. Growth exposed the cost structure.

| Criteria | Supabase Storage | Cloudflare R2 |
|---|---|---|
| Cost | Free tier limited, then steep | 10GB free, then $0.015/GB |
| Egress cost | Charged | **Free** |
| CDN | Supabase CDN | Cloudflare global CDN built-in |
| Custom domain | Limited | Full control |
| Presigned URL | Not supported | S3-compatible API, full support |

Egress cost was the deciding factor. A storybook service serves images repeatedly. Every page has an illustration. Users revisit their books. R2 charges zero egress. The savings compound with every new user.

### R2 Setup

| Environment | Bucket | Public URL |
|---|---|---|
| Development | `myapp-dev` | `cdn-dev.example.com` |
| Production | `myapp` | `cdn.example.com` |

Custom domains via Cloudflare DNS. No CORS headaches — same-origin requests through the custom domain.

### Directory Mapping

Directory structure mirrors the old Supabase buckets. This minimized migration complexity.

| Old Supabase Bucket | R2 Directory | Purpose |
|---|---|---|
| `media` | `media/` | Scene images, PDFs, previews |
| `shared` | `shared/` | Fonts, notice images |
| (new) | `assets/` | Character images |

### Presigned URL Workflow

The old system had clients upload via the Supabase SDK. R2 uses presigned URLs instead.

```
Client -> POST /api/upload/presign
          { bucket: "assets",
            path: "abc.jpg",
            contentType: "image/jpeg" }
       <- { uploadUrl: "https://...(signed URL)",
            publicUrl: "https://cdn.example.com/assets/abc.jpg" }

Client -> PUT uploadUrl (file binary)

Client -> Access via publicUrl
```

Three advantages:

1. **Backend never touches file bytes**: Bandwidth saved.
2. **Auth check at presign time**: Backend verifies the user before issuing the signed URL.
3. **No client SDK required**: Standard HTTP PUT handles the upload.

### Legacy URL Compatibility

Existing DB rows contain Supabase Storage URLs. Bulk replacement is risky. Gradual transition instead.

- **Existing Supabase URLs**: Keep working. Supabase Storage stays public.
- **New uploads**: Store R2 URLs.
- **Delete operations**: Only delete R2 URLs. Supabase URLs get a warning log.

Over time, R2 URL ratio increases naturally. Once coverage is sufficient, Supabase Storage gets decommissioned.

### Code Changes

| App | Change |
|---|---|
| Backend | `supabase-storage.ts` replaced by `r2-storage.ts` (same interface) |
| Scheduler | Supabase SDK removed, `uploadToR2()` function added |
| Admin | Direct Supabase upload replaced with presigned URL flow |
| Web | PDF generation uses presigned URLs |

`r2-storage.ts` maintains identical function signatures to `supabase-storage.ts`. `uploadImage`, `downloadImage`, `deleteFile` — change the import path and everything works.

---

## Part 2: Scheduler Modularization

### Problem: 638 Lines in One File

During R2 migration, the Scheduler's `src/index.ts` revealed its true size: 638 lines. DB schema definitions, lock management, Toss Payments API calls, automated billing, notice publishing, stale job recovery, job processing — all in one file.

Finding the storage code to replace meant scrolling through billing logic and lock management. The file needed splitting before any safe modification.

### Before

```
scheduler/src/
+-- constants.ts
+-- index.ts              # 638 lines -- everything
+-- services/
    +-- generation.service.ts
```

### After

```
scheduler/src/
+-- constants.ts
+-- index.ts              # ~100 lines -- orchestrator only
+-- db/
|   +-- schema.ts         # Table definitions
|   +-- client.ts         # DB connection factory
+-- utils/
|   +-- lock.ts           # Lock file management
+-- services/
    +-- payment-gateway.ts    # Payment API utilities
    +-- publisher.service.ts  # Scheduled publishing
    +-- billing.service.ts    # Automated billing
    +-- job-recovery.ts       # Stale job recovery
    +-- job.service.ts        # Job processing loop
    +-- generation.service.ts # AI generation pipeline
```

Eight module files. Each has a single responsibility. Each is testable in isolation.

### DB Client Factory Pattern

Module-scope variables create hidden dependencies. The factory pattern makes them explicit.

A factory function creates the DB client from a connection URL. The entry point creates one client instance and passes it into every service function as the first parameter. This is dependency injection at the function level — easy to mock for tests, no global state.

### Toss API Signature Change

Module-scope closure captured `TOSS_SECRET_KEY` implicitly. Explicit parameters replaced it.

Previously, functions captured secrets via module-scope closures (implicit dependency). After refactoring, every external dependency — including API keys — is passed as an explicit function parameter. Every dependency is visible in the function signature. No hidden state.

### index.ts: Orchestrator Only

After refactoring, `index.ts` does five things:

```
index.ts responsibilities:
1. Validate environment variables (DATABASE_URL required)
2. Acquire lock (prevent duplicate execution)
3. Register signal handlers (SIGTERM, SIGINT)
4. Create DB client
5. Register 3 timers:
   - 30s: Job processing
   - 60s: Notice publishing + job recovery
   - 1h:  Automated billing
```

No business logic. No API calls. No DB queries. Pure orchestration.

### Zero Behavior Change

Logic stayed identical. Only file locations changed. `tsup` bundles everything into a single output. Deployment is unaffected.

Build verification:

```bash
yarn build:scheduler  # compiles without error
# Deploy as usual
```

No new dependencies. No new APIs. No changed behavior. The refactor is invisible to users and downstream systems.

## Why Both on the Same Day

The R2 migration required changing storage code inside the Scheduler. The 638-line monolith made that change risky. Split first, then swap.

Refactoring gets postponed indefinitely if separated from feature work. Doing it when the pain is fresh — when you're staring at the 638-line file trying to find the upload function — that's the right time. The migration provided the motivation. The modularization provided the safety.

Next: content type expansion design.
