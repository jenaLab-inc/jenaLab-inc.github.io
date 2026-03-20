---
layout: post
ref: storida-system-architecture
title: "Storida System Architecture — Monorepo Design for AI SaaS"
date: 2026-02-03 10:00:00
categories: architecture
author: jenalab
image: /assets/article_images/storida/architecture.svg
image2: /assets/article_images/storida/architecture.svg
---

## Background

Enter a character name and a situation. Get a storybook. Claude writes the text. Gemini draws the illustrations. Storida automates that pipeline as an AI SaaS product.

This post covers the system architecture and the design decisions behind it.

## Why a Monorepo

Storida consists of 4 independent apps.

| App | Role | Tech | Port |
|---|---|---|---|
| **Web** | User-facing web app | Next.js + React | 3000 |
| **Admin** | Admin panel | Next.js + React | 3001 |
| **Backend** | API server | Express.js + TypeScript | 4000 |
| **Scheduler** | Batch processor | Node.js + TypeScript | — |

All four apps share DB schemas, types, and business logic. Splitting them into separate repos breaks type synchronization. Every schema change forces careful deployment ordering. Yarn Workspaces monorepo eliminates both problems.

```
storida/source/
├── apps/
│   ├── web/          # @storida/web
│   ├── admin/        # @storida/admin
│   ├── backend/      # @storida/backend
│   └── scheduler/    # @storida/scheduler
├── packages/
│   └── types/        # @storida/types (shared types)
└── package.json      # Yarn Workspaces root
```

Shared types live in `packages/types`. All 4 apps import them as `@storida/types`. Change a schema in one place. Build-time errors catch mismatches instantly.

## Architecture Layers

Four layers, top to bottom.

```
┌─────────────────────────────────────────────────┐
│              Presentation Layer                  │
│         Web (Next.js)  ·  Admin (Next.js)       │
└────────────────────┬────────────────────────────┘
                     │ HTTP + Firebase ID Token
┌────────────────────▼────────────────────────────┐
│              Application Layer                   │
│           Backend (Express + Drizzle)            │
│   Feature-based: books, characters, styles ...   │
└────────────────────┬────────────────────────────┘
                     │ Job Queue (DB polling)
┌────────────────────▼────────────────────────────┐
│              Processing Layer                    │
│         Scheduler (Node.js cron loop)            │
│   Claude API (text)  ·  Gemini API (images)      │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│              Infrastructure Layer                │
│  PostgreSQL (Supabase)  ·  Cloudflare R2         │
│  Firebase Auth  ·  Supabase Realtime             │
└─────────────────────────────────────────────────┘
```

### Presentation to Application

Web and Admin call the Backend API. Authentication uses Firebase ID Tokens in `Authorization: Bearer` headers. Backend middleware verifies them with `adminAuth.verifyIdToken()`.

### Application to Processing

Backend does not call AI directly. It writes a job to the `jobs` table and responds immediately. The Scheduler polls for PENDING jobs every 30 seconds.

The reason: Vercel enforces a 60-second timeout. AI generation takes 2-4 minutes. Synchronous processing is impossible.

### Processing to Infrastructure

The Scheduler calls Claude and Gemini APIs directly. Results go to PostgreSQL and Cloudflare R2. Supabase Realtime pushes progress updates to the Web client in real time.

## Feature-Based Directory Structure

Every app uses feature-based organization. Files group by domain, not by layer.

```
src/modules/
├── books/
│   ├── book.routes.ts       # Express routes
│   ├── book.service.ts      # Business logic
│   └── book.validation.ts   # Zod schemas
├── characters/
├── generation/
├── styles/
├── subscriptions/
└── ...
```

Adding a feature means all related files sit in one folder. Working on `subscriptions` requires no digging through other directories. Context switching drops.

## Tech Stack Decisions

| Area | Choice | Alternative | Reason |
|---|---|---|---|
| ORM | Drizzle | Prisma | SQL-first design, lightweight runtime bundle, Supabase compatibility |
| Validation | Zod | Joi, Yup | TypeScript type inference, runtime + type safety in one |
| UI | shadcn/ui | MUI, Ant Design | Copy-paste model for full customization, Tailwind-native |
| Image AI | Gemini Flash | DALL-E, SD | Cost-to-quality ratio, character consistency |
| Text AI | Claude Sonnet | GPT-4 | Creative writing quality, structured output reliability |

Drizzle ORM feels like writing SQL directly. Type safety stays intact. The combination of Drizzle `db:push` and Supabase `db diff` proved more predictable in production than Prisma's auto-migration.

## Deployment Architecture

```
CloudFlare (CDN + DNS)
    │
    ├── Vercel
    │   ├── Web (Next.js)      ← app.example.com
    │   └── Admin (Next.js)    ← admin.example.com
    │
    ├── Railway
    │   ├── Backend (Express)  ← api.example.com
    │   └── Scheduler (Node)   ← no external endpoint
    │
    ├── Supabase
    │   ├── PostgreSQL          ← DB
    │   └── Realtime            ← real-time events
    │
    └── Cloudflare R2
        └── cdn.example.com     ← image storage
```

Vercel hosts the frontends. Railway hosts Backend and Scheduler. The Scheduler has no external endpoint. It polls the database directly. Each service deploys independently and scales horizontally.

## Key Design Decisions

1. **Monorepo**: Yarn Workspaces for shared types and schema consistency
2. **Async Job Queue**: DB-based queue to bypass Vercel timeouts and control concurrency
3. **Scheduler calls AI directly**: No Backend intermediary. Better performance and fault isolation.
4. **Feature-based structure**: Domain-centric file organization for higher cohesion
5. **Supabase Realtime**: Real-time progress delivery without a dedicated WebSocket server

Next: the Drizzle ORM + Supabase database workflow.
