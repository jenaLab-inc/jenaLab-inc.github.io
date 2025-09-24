---
title: "Armo 개발 단계 #4: 콘텐츠 CRUD(초안·발행·예약) + 슬러그"
description: “artists, artworks, goods, packages, stories, journals, faqs 전 엔드포인트에 공통 상태머신(draft→scheduled|published)과 슬러그 자동생성/중복처리를 적용하고, API DoD(초안 생성→발행 업데이트)까지 한 사이클을 완주합니다.”
date: 2025-09-19 10:00:00 +0900
categories: [Armo, Backend, Fastify, Prisma]
tags: [CRUD, 상태머신, 예약발행, 슬러그, 유니크검증, artists, artworks, goods, packages, stories, journals, faqs, Fastify, Prisma, Node.js]
keywords: [“콘텐츠 CRUD”,“초안”,“발행”,“예약발행”,“슬러그”,“유니크”,“Fastify”,“Prisma”,“artists”,“artworks”,“goods”,“packages”,“stories”,“journals”,“faqs”]
author: "Jerome Na"
---

이 글은 Armo 개발 단계 #4, 콘텐츠를 초안 → 발행 → 예약발행까지 다루는 풀 플로우와 슬러그 전략을 순서대로 구현합니다.  
(스택: Fastify + Prisma + TypeScript)

---

## 목표
- 글/콘텐츠를 만들고(초안) → 예약/발행까지 한 사이클 가능
- 엔드포인트: artists/*, artworks/*, goods/*, packages/*, stories/*, journals/*, faqs/*
- 필수 필드 검증: 제목, 썸네일, 슬러그 유니크
- 상태머신: draft -> scheduled | published (+ publishedAt)
- 슬러그 자동 생성 및 중복 처리
- DoD: API로 내용 생성 → draft 저장 → published 업데이트 성공

---

## 1 Prisma 스키마 — 공통 상태/필드
모든 리소스(artists, artworks, goods, packages, stories, journals, faqs)에 동일한 최소 필드를 둡니다.

```ts
// prisma/schema.prisma
enum PublishStatus {
  DRAFT
  SCHEDULED
  PUBLISHED
}

model Artist {
  id           String        @id @default(cuid())
  title        String
  thumbnailUrl String
  slug         String        @unique
  status       PublishStatus @default(DRAFT)
  scheduledAt  DateTime?
  publishedAt  DateTime?
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt
}

// 동일 스펙으로 복제 — 리소스명만 변경
model Artwork   { ... }
model Good      { ... }
model Package   { ... }
model Story     { ... }
model Journal   { ... }
model Faq       { ... }
```

왜 전부 유니크 슬러그?: 페이지 라우팅·SEO·공유 URL 안정성을 위해 각 컬렉션 내 유니크를 보장합니다(요구사항에 맞춰 “전역 유니크”로 바꾸고 싶다면 별도 SlugRegistry 테이블로 통합 관리하세요).

마이그레이션:
```bash
pnpm prisma migrate dev --name stage4_content_crud_slug
```
---

## 2 Zod DTO — 필수 필드 검증
```ts
// src/dto/content.dto.ts
import { z } from 'zod'

export const createContentDto = z.object({
  title: z.string().trim().min(1, '제목은 필수입니다'),
  thumbnailUrl: z.string().url('썸네일은 URL이어야 합니다'),
  // slug는 서버에서 자동 생성
  // status는 생성 시 DRAFT 고정
})

export const updateContentDto = z.object({
  title: z.string().trim().min(1).optional(),
  thumbnailUrl: z.string().url().optional(),
})

export type CreateContentDto = z.infer<typeof createContentDto>
export type UpdateContentDto = z.infer<typeof updateContentDto>
```

---

## 3 슬러그 자동 생성 & 중복 처리
```ts
// src/utils/slug.ts
import slugify from 'slugify'

export function toBaseSlug(title: string) {
  const base = slugify(title, { lower: true, strict: true, trim: true, locale: 'ko' })
  return base || 'item'
}
```

```ts
// src/services/slug.service.ts
import { PrismaClient } from '@prisma/client'
import { toBaseSlug } from '../utils/slug'

const prisma = new PrismaClient()

// 리소스별 델리게이트를 받아 유니크 검사
export async function makeUniqueSlugFor(
  title: string,
  delegate: any,        // e.g. prisma.artist
  exceptId?: string,    // update 시 자기 자신 제외
) {
  const base = toBaseSlug(title)
  let candidate = base
  let n = 2

  // eslint-disable-next-line no-constant-condition
  while (true) {
    const found = await delegate.findFirst({
      where: {
        slug: candidate,
        ...(exceptId ? { NOT: { id: exceptId } } : {}),
      },
      select: { id: true },
    })
    if (!found) return candidate
    candidate = `${base}-${n++}`
  }
}
```

---

## 4 공통 서비스 팩토리 — 상태머신 포함
```ts
// src/services/content.factory.ts
import { PrismaClient, PublishStatus } from '@prisma/client'
import { makeUniqueSlugFor } from './slug.service'
import { CreateContentDto, UpdateContentDto } from '../dto/content.dto'

const prisma = new PrismaClient()

type Delegates =
  | PrismaClient['artist']
  ...

export function makeContentService(delegate: Delegates) {
  return {
    async createDraft(input: CreateContentDto) {
      const slug = await makeUniqueSlugFor(input.title, delegate)
      return delegate.create({
        data: { ...input, slug, status: 'DRAFT' },
      })
    },

    async update(id: string, input: UpdateContentDto) {
      const current = await delegate.findUniqueOrThrow({ where: { id } })
      ...
    },

    async schedule(id: string, whenIso: string) {
      const when = new Date(whenIso)
      ...
    },

    async publishNow(id: string) {
      return delegate.update({
        where: { id },
        data: { status: 'PUBLISHED', scheduledAt: null, publishedAt: new Date() },
      })
    },

    async list(status?: PublishStatus, q?: string) {
      return delegate.findMany({
        where: {
          ...(status ? { status } : {}),
          ...(q
            ? { OR: [{ title: { contains: q, mode: 'insensitive' } }] }
            : {}),
        },
        orderBy: [{ createdAt: 'desc' }],
      })
    },

    async remove(id: string) {
      await delegate.delete({ where: { id } })
    },
  }
}

// 각 리소스용 인스턴스
export const ArtistService  = makeContentService(prisma.artist)
...

```

---

## 5 라우트 — 7개 리소스 공통 패턴

엔드포인트 패턴(모두 동일):
- POST /api/{resource} — 초안 생성(draft)
- GET /api/{resource} — 목록(필터: status, q)
- GET /api/{resource}/:id — 단건 조회
- PATCH /api/{resource}/:id — 수정(발행 전에는 슬러그 자동 갱신)
- POST /api/{resource}/:id/schedule — 예약발행(scheduledAt)
- POST /api/{resource}/:id/publish — 즉시 발행
- DELETE /api/{resource}/:id — 삭제

```ts
// src/routes/content.routes.ts
import { FastifyInstance } from 'fastify'
import { z } from 'zod'
import {
  ArtistService, ArtworkService, GoodService,
  PackageService, StoryService, JournalService, FaqService
} from '../services/content.factory'
import { createContentDto, updateContentDto } from '../dto/content.dto'

const scheduleDto = z.object({ when: z.string().datetime() })

function mountContent(app: FastifyInstance, base: string, svc: any) {
  app.post(`/api/${base}`, async (req, rep) => {
    const input = createContentDto.parse(req.body)
    const data = await svc.createDraft(input)
    return rep.code(201).send(data)
  })

  app.get(`/api/${base}`, async (req) => {
    const { status, q } = (req.query as any) || {}
    return svc.list(status, q)
  })

  app.get(`/api/${base}/:id`, async (req) => {
    const { id } = req.params as any
    // findUniqueOrThrow로 교체하고 싶다면 서비스에 노출하세요
    return svc.update(id, {}) // 조회 전용 핸들러를 별도 두어도 됩니다
  })

  app.patch(`/api/${base}/:id`, async (req) => {
    const { id } = req.params as any
    const input = updateContentDto.parse(req.body)
    return svc.update(id, input)
  })

  app.post(`/api/${base}/:id/schedule`, async (req) => {
    const { id } = req.params as any
    const { when } = scheduleDto.parse(req.body)
    return svc.schedule(id, when)
  })

  app.post(`/api/${base}/:id/publish`, async (req) => {
    const { id } = req.params as any
    return svc.publishNow(id)
  })

  app.delete(`/api/${base}/:id`, async (req, rep) => {
    const { id } = req.params as any
    await svc.remove(id)
    return rep.code(204).send()
  })
}

export async function registerContentRoutes(app: FastifyInstance) {
  mountContent(app, 'artists',  ArtistService)
  ...
}
```

권한: publish/schedule/delete에는 관리자 가드를 붙이세요. (preHandler: [requireAdmin])
검증: 모든 POST/PATCH에서 Zod로 제목/썸네일 필수 검사, 슬러그 유니크는 DB + 사전조회로 이중 방지.

---

## 6 예약발행 스케줄러(공통)
```ts
// src/jobs/publish-scheduler.ts
import cron from 'node-cron'
import { PrismaClient, PublishStatus } from '@prisma/client'

const prisma = new PrismaClient()
const delegates = [
  prisma.artist, prisma.artwork, prisma.good,
  prisma.package, prisma.story, prisma.journal, prisma.faq,
]

export function startPublishScheduler() {
  // 매 10초마다 체크(데모). 운영은 1분 단위 권장.
  cron.schedule('*/10 * * * * *', async () => {
    const now = new Date()
    await Promise.all(delegates.map(async (d) => {
      const due = await d.findMany({
        where: { status: PublishStatus.SCHEDULED, scheduledAt: { lte: now } },
        select: { id: true },
        take: 100,
      })
      if (!due.length) return
      await prisma.$transaction(
        due.map(({ id }) => d.update({
          where: { id }, data: { status: PublishStatus.PUBLISHED, scheduledAt: null, publishedAt: now },
        }))
      )
    }))
  })
}
```
서버 부팅 시 구동:
```ts
// src/server.ts
import Fastify from 'fastify'
import { registerContentRoutes } from './routes/content.routes'
import { startPublishScheduler } from './jobs/publish-scheduler'

const app = Fastify({ logger: true })

app.register(registerContentRoutes)
startPublishScheduler()

app.listen({ port: 3000, host: '0.0.0.0' }).then(() => {
  app.log.info('Server http://localhost:3000')
})
```

---

## 7 상태머신 규칙 요약
- 생성시: status = DRAFT, slug = 자동생성(중복시 -2, -3 …)
- draft → scheduled: scheduledAt(미래) 필수, publishedAt=null
- scheduled → published: 스케줄러가 도달 시 자동 전환(publishedAt=now)
- draft → published: 즉시 발행 엔드포인트로 전환
- 발행 후 제목 수정: 기본적으로 슬러그 고정(링크 안정성). 발행 전(DRAFT/SCHEDULED)에는 제목 변경 시 슬러그 자동 갱신

---

## 8 DoD — 한 사이클 통과 시나리오
다음은 stories 리소스로 예시했지만, 7개 전부 동일합니다.
```bash
# 1) 초안 생성 (DRAFT)
http POST :3000/api/stories title='아르모 출범' thumbnailUrl='https://img.cdn/armo.png'
# -> 201 Created, body.id 저장
export ID=cl_xyz123

# 2) 예약 발행 (SCHEDULED)
http POST :3000/api/stories/$ID/schedule when='2025-09-19T12:00:00+09:00'
# -> status: SCHEDULED, scheduledAt: …

# 3) 즉시 발행으로 변경(선택)
http POST :3000/api/stories/$ID/publish
# -> status: PUBLISHED, publishedAt: now

# 4) 검증 — 목록/단건 조회
http GET :3000/api/stories status==PUBLISHED
http GET :3000/api/stories/$ID
```

DoD 통과 조건
- API로 내용 생성 → DB에 DRAFT 저장 확인
- 같은 항목을 SCHEDULED 또는 PUBLISHED로 업데이트 성공
- 슬러그가 자동 생성되고, 중복 시 접미(-2, -3 …) 처리 확인
- 필수 필드(제목/썸네일) 미제공 시 4xx 반환 확인

---

## 9 에러/엣지 처리 체크리스트

- 과거 시각 예약 금지 (400)
- 이미 PUBLISHED 항목 예약 금지
- 슬러그 충돌: 자동 접미 처리 + DB @unique로 이중 방어
- 발행 후 슬러그 변경 방지(링크 안정). 필요 시 별도 관리자 기능 + 301 리다이렉트 맵
- 스케줄러 주기/배치 크기/트랜잭션 적용(운영 트래픽↑ 시 큐/락 적용)

