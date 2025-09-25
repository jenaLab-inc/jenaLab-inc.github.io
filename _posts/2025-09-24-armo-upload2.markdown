---
title: “Armo 개발 단계 6 — 미디어 업로드 & 파생 생성(썸네일/웹용) 실전”
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

## 0. 설치 & 환경변수

```bash
pnpm add @supabase/supabase-js sharp pg-boss @fastify/multipart
```
.env (이미 일부 있을 수 있음)
```bash
DATABASE_URL=postgresql://USER:PASS@HOST:5432/DB
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJ... (서버 전용 키)
SUPABASE_BUCKET=media
```

- SUPABASE_URL은 SUPABASE에서 Project Setting의 API를 확인하면 주소가 나옴.
- SERVICE_ROLE_KEY? 서버에서 업로드/다운로드/업서트(생성·덮어쓰기)를 해야 하므로 권한이 필요.

---

## 1. Fastify 플러그인 등록 (multipart, boss, 에러형식)

src/index.ts

```ts
import Fastify from 'fastify'
import multipart from '@fastify/multipart'
import { registerMediaRoutes } from './routes/media.routes'
import { startDerivativeWorker } from './jobs/derivative.worker'

const app = Fastify({ logger: true })
await app.register(multipart, { limits: { fileSize: 30 * 1024 * 1024 } }) // 30MB

// (선택) 공통 에러형식 변환 미들웨어가 있다면 여기 등록

await registerMediaRoutes(app)

// MVP: API 프로세스에서 워커도 함께 가동 (운영은 분리 권장)
// startDerivativeWorker()

app.listen({ port: 3000, host: '0.0.0.0' })
```

---

## 2. 버킷 상수 & 검증 & Supabase 클라이언트 & pg-boss 싱글턴
src/lib/buckets.ts
```ts
export const BUCKETS = ['common', ...] as const
export type BucketName = typeof BUCKETS[number]

export function assertBucket(v: string): asserts v is BucketName {
  if (!BUCKETS.includes(v as any)) {
    throw new Error(`Unknown bucket: ${v}`)
  }
}
```

src/lib/supabase.ts

```ts
import { createClient } from '@supabase/supabase-js'

export const supa = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
  { auth: { persistSession: false } }
)
```

src/jobs/boss.ts (5 단계에서 만든 걸 재사용/개선)
- 안해도 됨.

```ts
import PgBoss from 'pg-boss'

let boss: PgBoss | null = null
export async function getBoss() {
  if (boss) return boss
  const b = new PgBoss({
    connectionString: process.env.DATABASE_URL!,
    schema: '잡 스키마 이름',
    application_name: 'armo-media-worker',
    newJobCheckInterval: 1000,
  })
  await b.start()
  boss = b
  const shutdown = async () => { try { await b.stop() } catch {} ; process.exit(0) }
  process.on('SIGINT', shutdown)
  process.on('SIGTERM', shutdown)
  return b
}
```

잡 이름 상수:
src/jobs/names.ts

```ts
export const JOB_GEN_DERIVATIVES = '미디어 잡 이름'
```

---

## 3. 업로드 라우트 (원본 업로드 + DB 기록 + 잡 enqueue)

src/routes/media.routes.ts
```ts
import { FastifyInstance } from 'fastify'
import { PrismaClient } from '@prisma/client'
import { supa } from '../lib/supabase'
import { randomUUID } from 'crypto'
import { getBoss } from '../jobs/boss'
import { JOB_GEN_DERIVATIVES } from '../jobs/names'
import { assertBucket } from '../lib/buckets'

const prisma = new PrismaClient()

export async function registerMediaRoutes(app: FastifyInstance) {
  app.post('/api/media/upload', async (req, rep) => {
    const file = await req.file()
    const { bucket } = req.query as any

    if (!file) return rep.code(400).send({ error: 'NO_FILE' })
    if (!bucket) return rep.code(400).send({ error: 'NO_BUCKET', message: 'bucket=artworks|goods|common 필요' })
    try { assertBucket(bucket) } catch (e: any) { return rep.code(400).send({ error: e.message }) }

    const ext = (file.filename.split('.').pop() || 'bin').toLowerCase()
    const now = new Date()
    const yyyy = now.getUTCFullYear()
    const mm = String(now.getUTCMonth() + 1).padStart(2, '0')
    const key = `폴더/${yyyy}/${mm}/${randomUUID()}.${ext}`

    // 1) 해당 버킷에 업로드
    const { error: upErr } = await supa.storage.from(bucket).upload(key, file.file, {
      contentType: file.mimetype, upsert: false
    })
    if (upErr) return rep.code(500).send({ error: 'UPLOAD_FAIL', message: upErr.message })

    // 2) DB 기록
    const asset = await prisma.mediaAsset.create({
      data: { bucket, path: key, kind: 'image' }
    })

    // 3) 파생 생성 잡 enqueue
    const boss = await getBoss()
    await boss.send(JOB_GEN_DERIVATIVES, { assetId: asset.id })

    return rep.code(201).send({ success: true, asset })
  })
}
```

---

## 4. 워커: Sharp로 파생 생성 (thumb/web/retina + WebP/AVIF) + 메타 저장

src/jobs/derivative.worker.ts

```ts
import { PrismaClient } from '@prisma/client';
import sharp from 'sharp';
import { supa } from '@/plugins/supabase';
import { JOB_GEN_DERIVATIVES } from './names';
import { getBossMedia } from '@/jobs/boss_media';

const prisma = new PrismaClient();

const VARIANTS = [
  { name: 'thumb', width: 480, format: 'jpg', quality: 82 },
  { name: 'web', width: 1280, format: 'webp', quality: 80 },
  { name: 'retina', width: 2560, format: 'avif', quality: 50 },
];

export async function startDerivativeWorker() {
  const boss = await getBossMedia();

  console.log('[generate-derivatives-worker] Starting worker...');

  await boss.work(JOB_GEN_DERIVATIVES, async (jobs) => {
    // v10에서는 항상 배열로 받음
    for (const job of jobs) {
      const { assetId } = job.data as { assetId: string };
      console.log(`[generate-derivatives-worker] Received ${assetId}`);
      const original = await prisma.mediaAsset.findUniqueOrThrow({ where: { id: assetId } });

      // 1) 원본 다운로드: 원본이 속한 버킷을 사용
      const { data, error } = await supa.storage.from(original.bucket).download(original.path);
      if (error || !data) throw new Error(`Download failed: ${error?.message}`);
      const origBuf = Buffer.from(await data.arrayBuffer());

      const normalized = await sharp(origBuf)
        .withMetadata({ exif: undefined })
        .toColorspace('srgb')
        .toBuffer();

      for (const v of VARIANTS) {
        const key = `폴더명/${v.name}/${assetId}-${v.width}.${v.format}`;
        
        let buffer: Buffer;
        let contentType: string;
        
        const resized = sharp(normalized).resize({ width: v.width, withoutEnlargement: true });
        
        // 포맷별 버퍼 생성
        switch (v.format) {
          case 'jpg':
            buffer = await resized.jpeg({ quality: v.quality, progressive: true }).toBuffer();
            contentType = 'image/jpeg';
            break;
          case 'webp':
            buffer = await resized.webp({ quality: v.quality }).toBuffer();
            contentType = 'image/webp';
            break;
          case 'avif':
            buffer = await resized.avif({ quality: v.quality }).toBuffer();
            contentType = 'image/avif';
            break;
          default:
            throw new Error(`Unsupported format: ${v.format}`);
        }
        
        const metadata = await sharp(buffer).metadata();
        
        // 스토리지 업로드
        await supa.storage
          .from(original.bucket)
          .upload(key, buffer, { contentType, upsert: true });
          
        // DB 기록
        await prisma.mediaAsset.create({
          data: {
            bucket: original.bucket,
            path: key,
            kind: '변형',
            width: metadata.width ?? null,
            height: metadata.height ?? null,
            sizeBytes: buffer.length,
            colorSpace: metadata.space ?? 'srgb',
            alt: original.alt,
          },
        });
        
        console.log(`[generate-derivatives-worker] Generated ${v.name} (${v.format}) for ${assetId}`);
      }
    }
  });
  console.log('[generate-derivatives-worker] started');
}

```

포인트
- idempotent 요구가 있으면: 업로드 전에 path 중복 조회 → 존재하면 skip.
- 큰 원본이라면 .resize({ withoutEnlargement: true })로 확대 방지.
-	화질 값은 운영 중 트래픽/품질 보고 조정.

---

## 5. 메타 저장 설계 (이미 MediaAsset 활용)

- 원본: kind = 'image', path = original/..., width/height/sizeBytes는 옵션(원본 메타를 저장하려면 업로드 직후 sharp로 한 번 읽어 추가해도 됨)
- 파생: kind = 'derivative', 각 포맷/사이즈를 별도 row로 기록
- 색공간: colorSpace='srgb'로 통일(나중에 품질 이슈 추적 용이)
- path는 유니크라 중복 삽입 방지

(원본 메타도 저장하고 싶다면 업로드 직후 아래처럼)

```ts
const probe = await sharp(await file.toBuffer()).metadata()
await prisma.mediaAsset.update({
  where: { id: asset.id },
  data: {
    width: probe.width ?? null,
    height: probe.height ?? null,
    sizeBytes: file.file.truncated ? null : file.file.readableLength ?? null,
    colorSpace: probe.space ?? null,
  }
})
```

---

## 6. 테스트 시나리오 (DoD 체크)

업로드 (httpie)
```bash
http -f POST :3000/api/media/upload bucket=버킷이름 file@./sample/hero.jpg
```
응답
```json
{ "success": true, "asset": { "id": "cl_xxx", "path": "original/2025/09/uuid.jpg", ... } }
```

10초 내 파생 생성 확인 (SQL)
```sql
-- 최근 생성된 파생 10개
select path, width, height, sizebytes, colorspace, createdat
from "미디어 테이블 이름"
where kind = 'derivative'
order by createdat desc
limit 10;
```

---

## 7.	Storage 확인 → /original/ + /폴더/ 폴더에 파일 생성

✅ DoD: 업로드 후 10초 내에 썸네일/웹/레티나 + WebP/AVIF 3종 이상 생성 & DB 메타 기록