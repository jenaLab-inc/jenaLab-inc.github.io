---
title: "Armo 개발 단계 #4: 콘텐츠 CRUD(초안·발행·예약) + 슬러그 실전"
description: “artists, artworks, goods, packages, stories, journals, faqs 전 엔드포인트에 공통 상태머신(draft→scheduled|published)과 슬러그 자동생성/중복처리를 적용하고, API DoD(초안 생성→발행 업데이트)까지 한 사이클을 완주합니다.”
date: 2025-09-24 10:00:00 +0900
categories: [Armo, Backend, Fastify, Prisma]
tags: [CRUD, 상태머신, 예약발행, 슬러그, 유니크검증, artists, artworks, goods, packages, stories, journals, faqs, Fastify, Prisma, Node.js]
keywords: [“콘텐츠 CRUD”,“초안”,“발행”,“예약발행”,“슬러그”,“유니크”,“Fastify”,“Prisma”,“artists”,“artworks”,“goods”,“packages”,“stories”,“journals”,“faqs”]
author: "Jerome Na"
---

이 글은 Armo 개발 단계 #4, 콘텐츠를 초안 → 발행 → 예약발행까지 다루는 풀 플로우와 슬러그 전략을 순서대로 구현한 실제 샘플입니다.  
(스택: Fastify + Prisma + TypeScript)

> 예시 소스는 전체 소스가 아닙니다. 구조만 보여주기 위한 소스이기 때문에 ...으로 되어 있는 부분은 각자에 맞게 수정하셔야 합니다.

---

## 준비 (1분 체크)

패키지: @prisma/client prisma zod slugify
```bash
pnpm add @prisma/client zod slugify
pnpm dlx prisma init
```

환경변수: DATABASE_URL 설정 (Supabase Postgres OK)

앞서 모델 스케치를 확인합니다.
```ts
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Status {
  draft
  scheduled
  published
}

model Artist {
  id        String   @id @default(cuid())
  slug      String   @unique
  ...
}

model Artwork {
  id        String   @id @default(cuid())
  slug      String   @unique
  ...
}

model Goods {
  id        String   @id @default(cuid())
  slug      String   @unique
  ...
}

...
```

---

## 1. 슬러그 유틸 (자동 생성 + 중복 처리)

- 소스 필드:
    - Artist → nameKo (fallback: nameEn)
    - Artwork → titleKo (fallback: titleEn)
    - Goods/Package/Story/Journal/Faq → nameKo/title/question 등 각 타입의 대표 텍스트
- 중복 처리: 같은 컬렉션 내에서 -2, -3… 접미

```ts
// src/utils/slug.ts
import slugify from 'slugify'

export function toBaseSlug(input: string) {
  const s = slugify(input ?? '', { lower: true, strict: true, trim: true, locale: 'ko' })
  return s || 'item'
}
```

```ts
// src/services/slug.service.ts
import { PrismaClient } from '@prisma/client'
import { toBaseSlug } from '../utils/slug'
const prisma = new PrismaClient()

type Delegate =
  | PrismaClient['artist'] | ...

export async function makeUniqueSlug(
  delegate: Delegate,
  baseText: string,
  exceptId?: string,
) {
  const base = toBaseSlug(baseText)
  let candidate = base
  let n = 2
  // eslint-disable-next-line no-constant-condition
  while (true) {
    const found = await delegate.findFirst({
      where: { slug: candidate, ...(exceptId ? { NOT: { id: exceptId } } : {}) },
      select: { id: true },
    })
    if (!found) return candidate
    candidate = `${base}-${n++}`
  }
}
```

---

## 2. DTO & 에러 포맷(필수 필드 검증)

요구사항의 “제목/썸네일/슬러그 유니크” 중 슬러그는 서버가 생성하고, 썸네일은 모델별 필드가 달라서 가능한 범위 내에서 강제합니다.
(현재 스키마에 썸네일 필드는 Goods의 thumbnail, Artwork의 heroImage만 존재)

```ts
// src/dto/content.dto.ts
import { z } from 'zod'

// 공통 상태값
export const StatusEnum = z.enum(['draft', 'scheduled', 'published'])

// 개별 타입별 필수값 정의(스키마 기준)
export const createArtistDto  = z.object({
  nameKo: z.string().min(1, 'nameKo는 필수입니다'),
  nameEn: z.string().optional(),
  tagline: z.string().optional(),
  bio: z.string().optional(),
  profile: z.string().url().optional(), // 스키마상 optional
})

export const createArtworkDto = z.object({
  ...
})

export const createGoodsDto   = z.object({
  ...
})

export const createPackageDto = z.object({
  ...
})

export const createStoryDto   = z.object({
  ...
})

export const createJournalDto = z.object({
  ...
})

export const createFaqDto     = z.object({
  ...
})

export const updateDto = z.record(z.any()) // 간단히: 부분 업데이트 허용
export const scheduleDto = z.object({
  publishAt: z.string().datetime('ISO8601 형식이어야 합니다'),
})
```

---

## 3. 공통 서비스 팩토리(상태머신 내장)

- 생성 시: status=draft, slug=자동 생성, publishAt=null
- 예약: draft → scheduled (미래 시간만 허용)
- 발행: draft|scheduled → published (publishAt=now())

```ts
// src/services/content.factory.ts
import { PrismaClient, Status } from '@prisma/client'
import { makeUniqueSlug } from './slug.service'

const prisma = new PrismaClient()

type Delegates =
  | PrismaClient['artist']  | ...

type CreateInput = Record<string, any>
type UpdateInput = Record<string, any>

export function makeContentService(
  delegate: Delegates,
  // 각 타입별 슬러그 소스 텍스트 선택 로직
  pickSlugText: (inputOrExisting: any) => string,
) {
  return {
    // CREATE (draft)
    async createDraft(input: CreateInput) {
      // 슬러그 생성
      const baseText = pickSlugText(input)
      const slug = await makeUniqueSlug(delegate, baseText)

      return delegate.create({
        data: { ...input, slug, status: Status.draft, publishAt: null },
      })
    },

    // UPDATE (발행 전 제목 바뀌면 슬러그도 갱신)
    async update(id: string, input: UpdateInput) {
      const current = await delegate.findUniqueOrThrow({ where: { id } })
      let payload: any = { ...input }

      // draft/scheduled일 때만 슬러그 재생성
      if (current.status !== Status.published) {
        const merged = { ...current, ...input }
        const newBase = pickSlugText(merged)
        if (newBase) {
          payload.slug = await makeUniqueSlug(delegate, newBase, id)
        }
      }
      return delegate.update({ where: { id }, data: payload })
    },

    // SCHEDULE
    async schedule(id: string, publishAtIso: string) {
      const when = new Date(publishAtIso)
      if (isNaN(when.getTime()) || when <= new Date()) {
        throw new Error('publishAt은 미래 시각이어야 합니다.')
      }
      const cur = await delegate.findUniqueOrThrow({ where: { id } })
      if (cur.status === Status.published) {
        throw new Error('이미 발행된 항목은 예약할 수 없습니다.')
      }
      return delegate.update({
        where: { id },
        data: { status: Status.scheduled, publishAt: when },
      })
    },

    // PUBLISH NOW
    async publishNow(id: string) {
      return delegate.update({
        where: { id },
        data: { status: Status.published, publishAt: new Date() },
      })
    },

    // READ
    async getOne(id: string) {
      return delegate.findUniqueOrThrow({ where: { id } })
    },
    async list(params: { status?: Status; q?: string }) {
      const { status, q } = params || {}
      return delegate.findMany({
        where: {
          ...(status ? { status } : {}),
          ...(q ? {
            OR: [
              { slug: { contains: q, mode: 'insensitive' } },
              // 대표 텍스트 컬럼을 추가로 필터하고 싶다면 라우트에서 덧붙여도 됨
            ],
          } : {}),
        },
        orderBy: [{ createdAt: 'desc' }],
      })
    },

    async remove(id: string) {
      await delegate.delete({ where: { id } })
    },
  }
}

// 각 리소스용 서비스 인스턴스
export const ArtistService  = makeContentService(prisma.artist,  x => x.nameKo || x.nameEn)
...
```

---

## 4. Fastify 라우트: 7개 리소스 공통 엔드포인트
```ts
// src/routes/content.routes.ts
import { FastifyInstance } from 'fastify'
import { z } from 'zod'
import {
  createArtistDto, ...
} from '../dto/content.dto'
import {
  ArtistService, ...
} from '../services/content.factory'

// 헬퍼: Zod 검증
function validate<T>(schema: z.ZodType<T>, data: any): T {
  return schema.parse(data)
}

function mount(app: FastifyInstance, base: string, svc: any, createSchema: z.ZodTypeAny) {
  app.post(`/api/${base}`, async (req, rep) => {
    ...
  })

  app.get(`/api/${base}`, async (req) => {
    const { status, q } = (req.query as any) || {}
    return svc.list({ status, q })
  })

  app.get(`/api/${base}/:id`, async (req) => {
    const { id } = req.params as any
    return svc.getOne(id)
  })

  app.patch(`/api/${base}/:id`, async (req) => {
    ...
  })

  app.post(`/api/${base}/:id/schedule`, async (req) => {
    ...
  })

  app.post(`/api/${base}/:id/publish`, async (req) => {
    ...
  })

  app.delete(`/api/${base}/:id`, async (req, rep) => {
    ...
  })
}

export async function registerContentRoutes(app: FastifyInstance) {
  mount(app, 'artists',  ArtistService,  createArtistDto)
  ...
}
```

서버 연결:
```ts
// src/server.ts 또는 index.ts
import Fastify from 'fastify'
import { registerContentRoutes } from './routes/content.routes'

const app = Fastify({ logger: true })
app.register(registerContentRoutes)

app.listen({ port: 3000, host: '0.0.0.0' }).then(() => {
  app.log.info('API: http://localhost:3000')
})
```

---

## 5. 상태머신 규칙(실행 관점 요약)

- 생성: status = draft, publishAt = null, slug = 자동 생성
- 예약: draft → scheduled (요청 publishAt은 미래만)
- 발행: draft|scheduled → published, publishAt = now()
- 발행 후 제목 변경 시 슬러그 고정(링크 안정성). (지금 팩토리는 published일 때 슬러그 갱신하지 않음)

---

## 6. DoD(완료기준) 시나리오 — stories로 예시

httpie 설치

```bash
# macOS (Homebrew)
brew install httpie

# Ubuntu / Debian
sudo apt-get install httpie

# Windows (scoop)
scoop install httpie
```
테스트
```bash
# 1) 초안 생성
http POST :3000/api/stories title='다람쥐 티타임' body='...' type='artwork'
# => 201 Created { id, slug, status:"draft" }

export ID=cl_xxx

# 2) 예약
http POST :3000/api/stories/$ID/schedule publishAt='2025-10-01T10:00:00+09:00'
# => status:"scheduled", publishAt:...

# 3) 즉시 발행
http POST :3000/api/stories/$ID/publish
# => status:"published", publishAt:now()

# 4) 목록/단건 확인
http GET :3000/api/stories status==published
http GET :3000/api/stories/$ID
```

"ISO8601 형식이어야 합니다" 오류 대응
```bash
# 서울 시간(+09:00)로 예약
http POST :8080/api/stories/ID-여기에/schedule publishAt='2025-10-01T10:00:00+09:00'

# UTC(Z)로 성공함.
http POST :8080/api/stories/ID-여기에/schedule publishAt='2025-10-01T01:00:00Z'
```

통과 기준 (DoD)
- API로 내용 생성 → draft 저장
- 같은 항목을 scheduled/published로 업데이트 성공
- 슬러그 자동 생성 + 중복 접미 처리(-2, -3 …)
- 필수 필드 누락 시 4xx (Zod 에러)

---

## 7. 실무 팁 & 선택적 개선

- 썸네일 강제: 요구사항상 “제목/썸네일/슬러그 유니크”가 필수라면,
- Artwork heroImage, Goods thumbnail는 create DTO에서 필수로 바꿔 강제 검증.
- Artist/Package/Story/Journal/Faq에도 썸네일 필드가 필요하면 스키마에 추가(권장: thumbnail 공통).
- 전역 유니크 슬러그: 모든 컬렉션 간 충돌까지 막으려면 SlugRegistry(type, refId, slug @unique) 테이블로 관리.
- 예약 자동 발행: Job Queue(pg-boss) 워커에서 status=scheduled AND publishAt<=now() → published 전환(5단계에서 붙임).
- 권한: publish/schedule/delete는 Owner/Editor 가드 필수.
- 검색 인덱스: published 전환 시 Typesense 인덱싱 트리거(워커에서 처리).