---
title: “Armo 개발 단계 6 — 미디어 업로드 & 파생 생성(썸네일/웹용)”
description: “Supabase Storage에 원본 이미지를 업로드한 뒤, Sharp 워커를 통해 썸네일·웹·레티나 파생본을 자동 생성하고 메타데이터를 DB에 기록하는 과정을 정리합니다.”
date: 2025-09-24 10:00:00 +0900
categories: [Armo, Backend, Fastify, Prisma, Media]
tags: [미디어업로드, 이미지파생, sharp, Supabase, Storage, Fastify, Prisma]
keywords: [“Supabase Storage”, “Sharp”, “이미지 썸네일”, “웹 최적화”, “AVIF 변환”, “이미지 파생”, “Fastify”, “Prisma”]
author: "Jerome Na"
---

Supabase Storage에 원본 이미지를 업로드한 뒤, Sharp 워커를 통해 썸네일·웹·레티나 파생본을 자동 생성하고 메타데이터를 DB에 기록하는 과정을 정리합니다.

---
## 목표
- 큰 이미지를 넣어도 웹에서 빠르게 뜨도록 자동 파생본 생성
- 원본은 Supabase Storage에 저장
- Sharp 워커가 파생 생성 (thumb/web/retina + WebP/AVIF)
- 메타데이터(DB 기록): 경로, 해상도, 용량, 색공간
- DoD: 업로드 후 10초 내 파생 파일 3종 생성 + DB 메타 기록

---

## 0. 왜 필요한가?

Armo 프로젝트는 작품 이미지 → 굿즈 → 스토리로 이어지는 흐름을 갖고 있습니다.

하지만 원본 이미지를 그대로 노출하면:
- 용량이 크고,
- 모바일에서 느리게 뜨며,
- SEO/웹 접근성 점수가 떨어집니다.

👉 따라서 파생본을 자동 생성해 최적화된 사이즈(WebP/AVIF)를 제공해야 합니다.

---

## 1. 업로드 API (Supabase Storage)

우선, 관리자가 Admin에서 원본 이미지를 업로드할 수 있어야 합니다.

```ts
// src/routes/media.routes.ts
import { FastifyInstance } from 'fastify'
import { createClient } from '@supabase/supabase-js'
import { PrismaClient } from '@prisma/client'
import { v4 as uuid } from 'uuid'

const prisma = new PrismaClient()
const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!, // 서버용
)

export async function registerMediaRoutes(app: FastifyInstance) {
  app.post('/api/media/upload', async (req, rep) => {
    const data = await req.file() // fastify-multipart 필요
    if (!data) return rep.code(400).send({ error: 'no file' })

    const ext = data.filename.split('.').pop()
    const key = `original/${uuid()}.${ext}`

    // 1. Supabase Storage에 업로드
    const { error } = await supabase.storage
      .from('media')
      .upload(key, data.file, { contentType: data.mimetype })
    if (error) throw error

    // 2. DB 기록 (원본만)
    const asset = await prisma.mediaAsset.create({
      data: { path: key, kind: 'image' },
    })

    // 3. 워커에 파생 생성 잡 등록
    await app.boss.send('워커 이름', { assetId: asset.id })

    return { success: true, asset }
  })
}
```

---

## 2. 파생 생성 워커 (Sharp)

Sharp 라이브러리를 사용해 원본 → 썸네일/웹/레티나 + WebP/AVIF로 변환합니다.

```ts
// src/jobs/derivative.worker.ts
import sharp from 'sharp'
import { PrismaClient } from '@prisma/client'
import { getBoss } from './boss'
import { createClient } from '@supabase/supabase-js'

const prisma = new PrismaClient()
const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
)

const SIZES = [
  { name: 'thumb', width: 480 },
  { name: 'web', width: 1280 },
  { name: 'retina', width: 2560 },
]

export async function startDerivativeWorker() {
  const boss = await getBoss()

  await boss.work('워커 이름', async (job) => {
    const { assetId } = job.data
    const asset = await prisma.mediaAsset.findUniqueOrThrow({ where: { id: assetId } })

    // 1. 원본 다운로드
    const { data, error } = await supabase.storage
      .from('media')
      .download(asset.path)
    if (error || !data) throw error

    const buffer = Buffer.from(await data.arrayBuffer())

    // 2. 각 사이즈별 파생 생성
    for (const { name, width } of SIZES) {
      const resized = await sharp(buffer).resize({ width }).toBuffer()
      const meta = await sharp(resized).metadata()

      const key = `폴더/파일 이름`
      await supabase.storage.from('media').upload(key, resized, {
        contentType: 'image/jpeg',
        upsert: true,
      })

      await prisma.미디어테이블이름.create({
        data: {
          path: key,
          kind: '폴더',
          width: meta.width,
          height: meta.height,
          sizeBytes: resized.length,
          colorSpace: meta.space,
          alt: asset.alt,
        },
      })

      // WebP
      const webp = await sharp(buffer).resize({ width }).webp().toBuffer()
      const webpKey = key.replace(/\.jpg$/, '.webp')
      await supabase.storage.from('media').upload(webpKey, webp, { contentType: 'image/webp', upsert: true })

      // AVIF
      const avif = await sharp(buffer).resize({ width }).avif().toBuffer()
      const avifKey = key.replace(/\.jpg$/, '.avif')
      await supabase.storage.from('media').upload(avifKey, avif, { contentType: 'image/avif', upsert: true })
    }

    console.log(`[폴더-worker] asset:${assetId} → derivatives generated`)
  })
}
```

## 3. DB 메타 기록

우리는 mediaAsset 테이블에 파생본도 기록합니다.
- path: Supabase Storage 키
- width, height, sizeBytes
- colorSpace: sRGB 확인용
- kind: "폴더"

이렇게 하면 웹 클라이언트는 DB 쿼리로 파생본 URL들을 한 번에 받아 상황에 맞는 사이즈를 로드할 수 있습니다.

## 4. 실행 및 테스트

1.	API 서버 실행 + 워커 실행
```bash
pnpm dev     # API
pnpm worker  # 워커 (pg-boss)
```

2.	Admin에서 이미지 업로드
```bash
POST /api/media/upload (multipart)
```

3.	DB 확인
```sql
select path, width, height, size_bytes, color_space
from 미디어테이블이름
where kind='폴더'
order by createdat desc;
```

4.	Storage 확인 → /original/ + /폴더/ 폴더에 파일 생성

✅ DoD: 업로드 후 10초 내에 썸네일/웹/레티나 + WebP/AVIF 3종 이상 생성 & DB 메타 기록