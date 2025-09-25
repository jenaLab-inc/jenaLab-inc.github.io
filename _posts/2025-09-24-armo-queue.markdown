---
title: “Armo 개발 단계 5. 잡 큐(pg-boss) & 예약 발행 워커”
description: “콘텐츠 예약 발행을 자동으로 처리하는 잡 큐(pg-boss) 워커를 구축합니다. 예약 시간에 맞춰 상태를 scheduled → published로 전환하고, 실패 시 재시도 및 데드레터 큐를 활용하는 방법을 소개합니다.”
date: 2025-09-23 10:00:00 +0900
categories: [Armo, Backend, Fastify, Prisma, Queue]
tags: [pg-boss, 잡큐, 예약발행, 워커, Fastify, Prisma, Node.js, 콘텐츠관리]
keywords: [“pg-boss”, “예약 발행”, “잡 큐”, “워크플로우”, “scheduled publish”, “콘텐츠 자동화”, “Fastify”, “Prisma”]
author: "Jerome Na"
---

콘텐츠 예약 발행을 자동으로 처리하는 잡 큐(pg-boss) 워커를 구축합니다. 예약 시간에 맞춰 상태를 scheduled → published로 전환하고, 실패 시 재시도 및 데드레터 큐를 활용하는 방법을 소개합니다.

## 목표
- 예약 시간에 맞춰 콘텐츠 자동 발행
- 실패 시 재시도 & 데드레터 큐 관리
- 워커 컨테이너 분리 가능 (API 내 스레드로 시작)

---

## 왜 잡 큐인가?

앞선 단계(4)에서 우리는 예약 발행 API까지만 구현했어요. 즉, 사용자는 status=scheduled, publishAt=future로 저장할 수 있지만, 시간이 지나도 자동 발행되지 않았습니다.

이를 해결하기 위해 **잡 큐(pg-boss)** 를 도입합니다.
- 예약된 시간에 맞춰 scheduled → published 상태 전환
- 작업 실패 시 자동 재시도 & 데드레터 큐(logs) 보관
- 워커를 API와 분리해도 되고, MVP 단계에서는 API 안에 스레드로 돌려도 충분

---

## 1. pg-boss 설치 및 연결
```bash
pnpm add pg-boss
```
.env
```env
DATABASE_URL=postgresql://user:pass@host:5432/db
```

---

## 2. 워커 초기화

```ts
// src/jobs/boss.ts
import PgBoss from 'pg-boss'

export async function createBoss() {
  const boss = new PgBoss({
    connectionString: process.env.DATABASE_URL!,
    schema: '잡 스키마 이름', // 전용 스키마 권장
    application_name: 'armo-worker',
  })
  await boss.start()
  return boss
}
```

마이그레이션은 await boss.start() 시 자동 실행됩니다.

---

## 3. 예약 발행 잡 정의

```ts
// src/jobs/publish.worker.ts
import { PrismaClient, Status } from '@prisma/client'
import { createBoss } from './boss'

const prisma = new PrismaClient()
const JOB_NAME = '잡 이름'

export async function startPublishWorker() {
  const boss = await createBoss()

  // 1. 워커 구독
  await boss.work(JOB_NAME, async (job) => {
    const { model, id } = job.data as { model: string; id: string }
    const delegate = (prisma as any)[model]
    if (!delegate) throw new Error(`Unknown model: ${model}`)

    const now = new Date()
    await delegate.update({
      where: { id },
      data: { status: Status.published, publishAt: now },
    })
    console.log(`[publish-worker] ${model}:${id} → published @${now.toISOString()}`)
  })

  console.log(`[publish-worker] started`)
}
```

---

## 4. 예약 잡 등록 API

콘텐츠를 예약할 때, 단순히 DB에 status=scheduled로 저장하는 대신 잡 큐에 푸시합니다.

```ts
// src/services/schedule.service.ts
import { createBoss } from '../jobs/boss'
const JOB_NAME = '잡 이름'

export async function enqueuePublish(model: string, id: string, when: Date) {
  const boss = await createBoss()
  await boss.schedule(JOB_NAME, when, { model, id })
}
```

주의: boss.schedule()은 DB에 “예약된 잡”으로 기록되고, when 시간이 되면 워커가 꺼내 실행합니다.

---

## 5. API 연동

예약 API에서 DB 저장 + 큐 등록을 동시에 실행합니다.

```ts
// 예: StoryService.schedule()
async schedule(id: string, publishAtIso: string) {
  const when = new Date(publishAtIso)
  if (isNaN(when.getTime()) || when <= new Date()) {
    throw new Error('publishAt은 미래 시각이어야 합니다.')
  }
  const story = await prisma.story.update({
    where: { id },
    data: { status: Status.scheduled, publishAt: when },
  })
  await enqueuePublish('story', id, when)
  return story
}
```

---

## 6. 실패 처리

- 재시도: pg-boss 기본값으로 실패 시 3회 재시도
- 데드레터 큐: 여전히 실패하면 failed 테이블에 기록 → 로그 분석 가능
- 로깅: console.log 대신 pino 같은 로거로 통합 추천

---

## 7. 실행
API 컨테이너와 별도 워커 컨테이너를 둘 수 있어요.

단일 프로세스 (MVP)
```ts
// src/server.ts
import { startPublishWorker } from './jobs/publish.worker'

// API 서버 시작 후 워커도 함께 실행
startPublishWorker()
```

분리 프로세스 (운영)
- fly.toml에서 api와 worker를 별도 프로세스로 정의
- worker.ts 엔트리포인트에서 startPublishWorker()만 실행

---

## 8. DoD 검증

- API로 status=scheduled, publishAt=1~2분 뒤 저장
- pg-boss 잡 큐에 스케줄링 확인 (job_queue.job 테이블에서 조회 가능)
- 1~2분 후 워커 로그에서 자동 발행 처리 확인

```bash
[publish-worker] story:cl_123 → published @2025-09-23T10:30:00.123Z
```

- DB 조회 시 status=published로 전환됨

✅ DoD: 예약 발행이 자동 상태 전환됨을 로그로 검증

