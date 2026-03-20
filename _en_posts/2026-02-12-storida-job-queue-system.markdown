---
layout: post
ref: storida-job-queue-system
title: "DB-Based Job Queue — Async Generation System to Bypass Vercel Timeouts"
date: 2026-02-12 10:00:00
categories: architecture
author: jenalab
image: /assets/article_images/storida/job-queue.svg
image2: /assets/article_images/storida/job-queue.svg
---

## The Problem: 2-4 Minutes of AI Generation

Claude writes text. 30 seconds to 1 minute. Gemini draws images. 20-30 seconds per scene, 2 minutes for 5 scenes. Total: 2-4 minutes per book.

Three problems followed.

1. **Vercel 60-second limit**: Web and Backend deploy on Vercel/Railway. API timeout is 60 seconds.
2. **No concurrency control**: Burst requests flood AI API calls. Server overload.
3. **No user feedback**: Users stare at a blank screen for 2-4 minutes.

The old system had Backend call AI directly and hold the response until completion. We switched to a DB-based job queue.

## Architecture Decision

### Options Considered

| Option | Pros | Cons |
|---|---|---|
| Scheduler calls AI directly | Best performance, independence, scalability | Possible code duplication |
| Scheduler calls Backend HTTP then AI | Code reuse | Network latency, Backend load |
| Scheduler imports Backend functions | Single codebase | Requires monorepo restructuring |

Scheduler calls AI APIs directly. Routing through Backend adds an unnecessary network hop. Backend failure blocks generation. Direct AI calls from Scheduler keep generation independent of Backend health.

## Job Queue Design

### Jobs Table

Each job record tracks its type (e.g., book generation or image regeneration), current status, the owning user, the target book, and retry metadata (attempt count, max attempts). A JSONB payload field carries all data the Scheduler needs for generation, eliminating additional DB lookups. Error messages and timestamps round out the record for observability.

### Status Flow

```
PENDING ──────> PROCESSING ──────> COMPLETED
                    |
                    | error
                    v
              attempts < 3?
              |-- Yes -> PENDING (retry)
              +-- No  -> FAILED
```

### Index Design

A composite index on job type and status is the critical one — the Scheduler's main query filters by type and status together. Additional indexes on user and book references support lookup and join operations.

## Backend: Respond Immediately

```
// POST /api/generate
// 1. Validate tokens and pre-deduct the expected cost
// 2. Create an empty book record with PENDING status
// 3. Register a job with the generation payload
// 4. Respond immediately (<1s) with book ID, job ID, and status
```

The user gets a response within 1 second and returns to the main screen. Generation runs in the background.

Token pre-deduction happens at request time. This prevents double-spending during the 2-4 minute generation window. If generation fails, the system refunds tokens.

## Scheduler: Job Processing Loop

```
// Every 30 seconds:
// 1. Query up to 3 oldest PENDING book-generation jobs
// 2. Process all fetched jobs in parallel
//    (each job runs independently — one failure does not affect others)
```

Concurrency caps at 3. This number balances AI API rate limits and server resources. `Promise.allSettled` ensures one job failure does not affect others.

### AI Generation Pipeline

```
// AI Generation Pipeline:
// 1. Prepare generation context from the job payload
// 2. Generate full text via Claude, update progress to WRITING
// 3. Parse the text into pages and save them
// 4. For each page sequentially:
//    - Generate an image via Gemini and upload it
//    - Update progress (IMAGES: page N of total)
// 5. Mark the book as completed
```

Image generation runs sequentially per page. Parallel calls trigger frequent 429 errors from Gemini. Sequential processing with per-page progress updates is the trade-off.

## Real-Time Progress: Supabase Realtime

The Web client subscribes to `contents` table changes via Supabase Realtime.

The Web client opens a Supabase Realtime channel that listens for row-level changes on the contents table, filtered to the current user. When the Scheduler updates a book's generation progress, the client receives the new progress value in real time and updates the UI.

### Polling Fallback

WebSocket connections drop. Missed events leave the UI stale. A 10-second polling interval runs as insurance. It activates only when books are generating.

When any book is in a generating state, a 10-second polling interval activates as a fallback. Once no books are generating, the polling stops automatically. This ensures the UI stays current even if the WebSocket connection drops.

## Stale Job Recovery

When the Scheduler restarts, `PROCESSING` jobs become orphaned. The system auto-recovers stale jobs on startup.

```
// On Scheduler startup:
// Find all jobs stuck in PROCESSING for over 30 minutes
// Reset their status back to PENDING for the next polling cycle to pick up
```

Any job stuck in `PROCESSING` for over 30 minutes resets to `PENDING`. The next polling cycle picks it up. Combined with max 3 attempts, this ensures no job is permanently lost.

## Performance Comparison

| Metric | Before (Synchronous) | After (Async Job Queue) |
|---|---|---|
| Request to response | 2-4 min | **< 1 second** |
| Concurrency | Unlimited (overload risk) | **Max 3 (controlled)** |
| User feedback | None | **Real-time progress** |
| Server failure impact | Entire generation fails | **Auto-retry recovers** |
| API timeout | Possible | **Impossible** |

## Why Not Redis or RabbitMQ

An external message queue adds infrastructure management overhead. At tens to hundreds of jobs per day, PostgreSQL's `SELECT ... ORDER BY created_at LIMIT 3` is sufficient. If traffic grows 10x, we introduce a dedicated queue then.

The database is already there. The polling interval is 30 seconds. The concurrency cap is 3. For this scale, a separate queue service is unnecessary complexity.

Next: the multi-character system and Claude Vision image analysis.
