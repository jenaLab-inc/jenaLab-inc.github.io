---
title: “Armo 개발 단계 #4: 콘텐츠 CRUD(초안·발행·예약) + 슬러그, 실전 가이드”
description: “Fastify + Prisma 기반으로 콘텐츠 CRUD 전 과정(초안, 발행, 예약발행)과 SEO 친화적 슬러그 전략을 순서대로 구현합니다.”
date: 2025-09-19 10:00:00 +0900
categories: [Armo, Backend, Fastify, Prisma]
tags: [콘텐츠관리, CRUD, 초안, 발행, 예약발행, 슬러그, Fastify, Prisma, Node.js, SEO]
keywords: [“콘텐츠 CRUD”, “초안”, “발행”, “예약 발행”, “슬러그”, “SEO”, “Fastify”, “Prisma”, “Node.js 블로그”]
author: "Jerome Na"
---

이 글은 Armo 개발 단계 #4, 콘텐츠를 초안 → 발행 → 예약발행까지 다루는 풀 플로우와 SEO 최적화 슬러그 전략을 순서대로 구현합니다.  
(스택: Fastify + Prisma + TypeScript)

---

## 목표
- 모델/마이그레이션 설계 (초안/발행/예약 상태, 시간 필드)
- SEO 슬러그 생성·중복 처리·갱신 규칙
- CRUD API (Create/Read/Update/Delete)
- 상태 전이(초안↔발행, 예약→자동발행)
- 예약발행 스케줄러(간단 Cron/Job)
- 실전 테스트 시나리오 (curl/HTTPie)

---

## 1 데이터 모델 설계
```ts
// prisma/schema.prisma
enum PostStatus {
  DRAFT
  SCHEDULED
  PUBLISHED
  ARCHIVED
}

model Post {
  id           String     @id @default(cuid())
  title        String
  content      String     @db.Text
  status       PostStatus @default(DRAFT)

  // SEO
  slug         String
  // 시간/상태 관리
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @updatedAt
  publishedAt  DateTime?
  scheduledAt  DateTime?
  ...
}
```

설계 포인트
- status 4단계: DRAFT → (SCHEDULED) → PUBLISHED → ARCHIVED
- scheduledAt 도달 시 자동 발행
- slug 전역 유니크(+ 충돌 시 -2, -3… 접미)

마이그레이션:
```bash
pnpm prisma migrate dev --name init_posts
```

---

## 2 유틸: 슬러그 생성 & 충돌 해결
```ts
// src/utils/slug.ts
import { PrismaClient } from '@prisma/client'
import slugify from 'slugify'

const prisma = new PrismaClient()

export async function makeUniqueSlug(title: string, hintId?: string) {
  const base = slugify(title, {
    lower: true,
    strict: true, // 특수문자 제거
    trim: true,
    locale: 'ko',
  }) || 'post'

  // 같은 제목이라도 같은 글이면 유지(hintId로 자기 자신 업데이트 허용)
  let candidate = base
  let n = 2

  // 충돌 검사
  // (hintId가 있으면 자기 자신은 무시)
  // eslint-disable-next-line no-constant-condition
  while (true) {
    const found = await prisma.post.findFirst({
      where: { slug: candidate, ...(hintId ? { NOT: { id: hintId } } : {}) },
      select: { id: true },
    })
    if (!found) return candidate
    candidate = `${base}-${n++}`
  }
}
```
규칙: 제목 수정 시 slug는 **발행 전(=DRAFT/SCHEDULED)** 에만 자동 갱신. 이미 PUBLISHED면 기본적으로 고정(링크 안정성). 필요 시 별도 수동 변경 엔드포인트 제공.

---

## 3 서비스 레이어: 상태 전이 보호 규칙
```ts
// src/services/post.service.ts
import { PrismaClient, PostStatus } from '@prisma/client'
import { makeUniqueSlug } from '../utils/slug'

const prisma = new PrismaClient()

export const PostService = {
  async createDraft(input: { title: string; content: string; authorId?: string }) {
    const slug = await makeUniqueSlug(input.title)
    return prisma.post.create({
      data: {
        ...input,
        status: PostStatus.DRAFT,
        slug,
      },
    })
  },

  async update(id: string, input: { title?: string; content?: string }) {
    ...
  },

  async publishNow(id: string) {
    const post = await prisma.post.findUniqueOrThrow({ where: { id } })
    if (post.status === PostStatus.PUBLISHED) return post
    return prisma.post.update({
      where: { id },
      data: { status: PostStatus.PUBLISHED, publishedAt: new Date(), scheduledAt: null },
    })
  },

  async schedule(id: string, when: Date) {
    ...
  },

  async unpublish(id: string) {
    ...
  },

  async remove(id: string) {
    return prisma.post.delete({ where: { id } })
  },

  async list(params: { status?: PostStatus; q?: string }) {
    const { status, q } = params
    return prisma.post.findMany({
      where: {
        ...(status ? { status } : {}),
        ...(q
          ? {
              OR: [
                { title: { contains: q, mode: 'insensitive' } },
                { content: { contains: q, mode: 'insensitive' } },
              ],
            }
          : {}),
      },
      orderBy: [{ createdAt: 'desc' }],
    })
  },
}
```

---

## 4 라우트: CRUD + 상태/예약 API
```ts
// src/routes/post.routes.ts
import { FastifyInstance } from 'fastify'
import { PostService } from '../services/post.service'

export async function registerPostRoutes(app: FastifyInstance) {
  app.post('/api/posts', async (req, rep) => {
    const { title, content } = req.body as any
    const post = await PostService.createDraft({ title, content, authorId: (req.user as any)?.id })
    return rep.code(201).send(post)
  })

  app.get('/api/posts', async (req) => {
    const { status, q } = (req.query as any) || {}
    return PostService.list({ status, q })
  })

  app.get('/api/posts/:id', async (req) => {
    const { id } = req.params as any
    return PostService.update(id, {}) // 조회 전용: findUnique 별도 구현해도 OK
  })

  app.patch('/api/posts/:id', async (req) => {
    ...
  })

  app.post('/api/posts/:id/publish', async (req) => {
    ...
  })

  app.post('/api/posts/:id/schedule', async (req) => {
    ...
  })

  app.post('/api/posts/:id/unpublish', async (req) => {
    ...
  })

  app.delete('/api/posts/:id', async (req, rep) => {
    ...
  })
}
```

보안: 발행/예약/삭제는 관리자 권한 가드 미들웨어를 붙여주세요. (e.g., preHandler: [requireAdmin])

---

## 5 예약발행 스케줄러

가볍게 시작하려면 node-cron으로 1분마다 폴링:
```ts
// src/jobs/publish-scheduler.ts
import cron from 'node-cron'
import { PrismaClient, PostStatus } from '@prisma/client'

const prisma = new PrismaClient()

export function startPublishScheduler() {
  // 매 분 0초마다
  cron.schedule('0 * * * * *', async () => {
    const now = new Date()
    const toPublish = await prisma.post.findMany({
      where: {
        status: PostStatus.SCHEDULED,
        scheduledAt: { lte: now },
      },
      select: { id: true },
      take: 50, // 배치 크기
    })

    if (!toPublish.length) return

    await prisma.$transaction(
      toPublish.map(({ id }) =>
        prisma.post.update({
          where: { id },
          data: { status: PostStatus.PUBLISHED, publishedAt: now, scheduledAt: null },
        }),
      ),
    )
    // TODO: 캐시 무효화/웹훅/검색 인덱싱 등
  })
}
```

서버 시작 시 활성화:
```ts
// src/server.ts
import Fastify from 'fastify'
import { registerPostRoutes } from './routes/post.routes'
import { startPublishScheduler } from './jobs/publish-scheduler'

const app = Fastify({ logger: true })

app.register(registerPostRoutes)

startPublishScheduler()

app.listen({ port: 3000, host: '0.0.0.0' }).then(() => {
  app.log.info('Server running on http://localhost:3000')
})
```

트래픽이 커지면 pg-boss/큐로 전환하고, 예약 도달 시점에 **단일 UPDATE … WHERE status=SCHEDULED AND scheduledAt<=now()** 로 아토믹 처리 + 락 전략을 권장합니다.

---

## 6 상태 전이 규칙 (요약)

- DRAFT → PUBLISHED: publishNow()
- DRAFT → SCHEDULED: schedule(when)
- SCHEDULED → PUBLISHED: 스케줄러 자동 전환
- PUBLISHED → DRAFT: unpublish() (링크 변동 주의)
- PUBLISHED 상태에서 제목 변경 시 기본 슬러그 고정(링크 안정). 정말 바꾸려면 관리 전용 엔드포인트 + 301 리다이렉트 맵을 마련합니다.

---

## 7 테스트 시나리오 (HTTPie 예시)
```bash
# 1) 초안 생성
http POST :3000/api/posts title='안녕, 아르모' content='콘텐츠 본문입니다.'

# → 응답의 id 복사
export ID=clzw...abc

# 2) 초안 수정 (발행 전이라 slug 자동 갱신)
http PATCH :3000/api/posts/$ID title='아르모, 첫 인사' content='업데이트 본문'

# 3) 예약 발행
http POST :3000/api/posts/$ID/schedule when='2025-09-19T10:30:00+09:00'

# 4) 목록 (예약 상태만)
http GET :3000/api/posts status==SCHEDULED

# 5) 즉시 발행으로 변경 (예약 취소 + 발행)
http POST :3000/api/posts/$ID/publish

# 6) 발행 취소 → 초안
http POST :3000/api/posts/$ID/unpublish

# 7) 삭제
http DELETE :3000/api/posts/$ID
```

---

## 8 에러/엣지 케이스 가이드

- 슬러그 충돌: 자동 -2, -3… 접미. 다국어(한글) 제목은 slugify(locale:'ko')로 처리.
- 예약시간 과거: 400 (유효성 검사).
- 이미 발행된 글 예약: 409/400 (비즈니스 규칙 위반).
- 발행 취소: 공개 URL 깨짐 위험 → UI에 경고/확인 모달 + 301 리다이렉트 옵션 제공.
- 제목 대량 변경: 발행 글은 기본 고정. 운영 중 URL 안정성 최우선.

---

## 9 SEO 체크리스트

- 슬러그는 짧고 의미 있는 키워드로 (불필요 stopword 제거).
- 발행 시점에 publishedAt → Sitemap/Feed 반영.
- 예약발행은 정확한 시간에 공개되어 SNS/알림과 동기화.
- 발행 후 Open Graph/Twitter 카드 메타 업데이트.
- 301 리다이렉트 맵 보유(슬러그 변경 시).

