---
layout: post
ref: storida-job-queue-system
title: "DB 기반 Job Queue — Vercel 타임아웃을 우회하는 비동기 생성 시스템"
date: 2026-02-12 10:00:00
categories: architecture
author: jenalab
image: /assets/article_images/storida/job-queue.svg
image2: /assets/article_images/storida/job-queue.svg
---

## 문제: 2~4분 걸리는 AI 생성

Claude가 텍스트를 쓴다. 30초~1분. Gemini가 이미지를 그린다. 장면당 20~30초, 5장면이면 2분. 전체 2~4분이 걸린다.

문제가 세 가지였다.

1. **Vercel 60초 제한**: Web과 Backend가 Vercel/Railway에 배포되는데, API 타임아웃이 60초
2. **동시 처리 제어 없음**: 요청이 몰리면 AI API 호출이 폭증하여 서버 과부하
3. **사용자 피드백 부재**: 2~4분간 빈 화면을 보며 대기

기존에는 Backend가 AI를 직접 호출하고 완료까지 응답을 보류했다. DB 기반 Job Queue로 전환했다.

## 아키텍처 결정

### 고려한 옵션

| 옵션 | 장점 | 단점 |
|---|---|---|
| Scheduler → AI 직접 호출 | 최고 성능, 독립성, 확장성 | 코드 중복 가능성 |
| Scheduler → Backend HTTP → AI | 코드 재사용 | 네트워크 레이턴시, Backend 부하 |
| Scheduler → Backend 함수 import | 단일 코드베이스 | 모노레포 구조 변경 필요 |

Scheduler가 AI API를 직접 호출한다. Backend를 경유하면 불필요한 네트워크 홉이 추가된다. Backend 장애 시 생성도 멈춘다. Scheduler가 독립적으로 AI를 호출하면 Backend 장애와 무관하게 작동한다.

## Job Queue 설계

### jobs 테이블

Job 레코드는 유형(BOOK_GENERATION, IMAGE_REGEN 등), 상태, 사용자/콘텐츠 참조, 생성 데이터(JSONB payload), 에러 메시지, 시도 횟수(기본 최대 3회), 타임스탬프를 저장한다. 상태는 PENDING → PROCESSING → COMPLETED 또는 FAILED로 전이한다.

`payload`에 생성 데이터를 JSONB로 저장한다. Scheduler는 추가 DB 조회 없이 payload만으로 작업을 수행한다.

### 상태 흐름

```
PENDING ──────▶ PROCESSING ──────▶ COMPLETED
                    │
                    │ 에러 발생
                    ▼
              attempts < 3?
              ├── Yes → PENDING (재시도)
              └── No  → FAILED
```

### 인덱스 설계

Scheduler가 유형 + 상태로 대기 중인 Job을 조회하므로, 이 두 컬럼의 복합 인덱스가 핵심이다. 사용자별, 콘텐츠별 조회를 위한 개별 인덱스도 추가했다.

## Backend: 요청 즉시 응답

```
// POST /api/generate
// 1. 인증된 사용자 확인 + 토큰 선차감
// 2. 빈 콘텐츠 레코드를 PENDING 상태로 생성
// 3. Job 레코드를 등록
// 4. 즉시 ID와 상태 반환 (<1초)
```

사용자는 1초 이내에 응답을 받고, 메인 화면으로 돌아간다. **생성은 백그라운드에서 진행**된다.

## Scheduler: Job 처리 루프

```
// 30초 간격으로 실행
// 1. PENDING 상태의 Job을 생성 순서대로 최대 3개 조회
// 2. 조회된 Job을 병렬 처리 (Promise.allSettled)
//    — 한 Job이 실패해도 다른 Job에 영향 없음
```

동시 처리 수를 3으로 제한한다. AI API rate limit과 서버 리소스를 고려한 수치다. `Promise.allSettled`를 써서 한 Job이 실패해도 다른 Job에 영향을 주지 않는다.

### AI 생성 파이프라인

```
// AI 생성 파이프라인
// 1. 생성 컨텍스트 준비 (payload에서 필요한 정보 추출)
// 2. Claude로 텍스트 생성 → 진행률 업데이트
// 3. 생성된 텍스트를 페이지별로 파싱 → DB 저장
// 4. Gemini로 페이지별 이미지 순차 생성 → 매 페이지마다 진행률 업데이트
// 5. 완료 처리
```

이미지 생성은 페이지별 순차 처리한다. 병렬로 돌리면 Gemini API의 429 에러가 빈발했다. 순차 처리하되, 각 이미지 생성 후 진행률을 업데이트한다.

## 실시간 진행률 — Supabase Realtime

Web은 Supabase Realtime으로 `contents` 테이블의 변경을 실시간 감지한다.

```
// Web: Supabase Realtime으로 콘텐츠 테이블 변경 감지
// 1. 해당 사용자의 콘텐츠 UPDATE 이벤트를 구독
// 2. 변경 발생 시 진행률 UI 업데이트
```

### 폴링 보조 전략

WebSocket 연결이 끊기면 이벤트를 놓친다. 보험으로 10초 간격 폴링을 병행한다. 생성 중인 책이 있을 때만 활성화한다.

```
// 폴링 보조 전략
// 생성 중인 콘텐츠가 있을 때만 10초 간격 폴링 활성화
// 모든 생성이 완료되면 폴링 자동 중지
```

## 성능 비교

| 지표 | 기존 (동기) | 개선 (비동기 Job Queue) |
|---|---|---|
| 요청 → 응답 | 2~4분 | **< 1초** |
| 동시 처리 | 무제한 (과부하 위험) | **최대 3개** (제어 가능) |
| 사용자 피드백 | 없음 | **실시간 진행률** |
| 서버 장애 영향 | 생성 전체 실패 | **재시도로 자동 복구** |
| API 타임아웃 | 발생 가능 | **발생 불가** |

## Stale Job 복구

Scheduler가 재시작되면 `PROCESSING` 상태인 Job이 방치될 수 있다. 시작 시 stale job을 자동 복구한다.

```
// Stale Job 복구
// 30분 이상 PROCESSING 상태로 방치된 Job을 PENDING으로 되돌린다
// Scheduler 시작 시 자동 실행
```

## 왜 Redis/RabbitMQ를 사용하지 않았나

외부 메시지 큐를 도입하면 인프라 관리 포인트가 늘어난다. 일 수십~수백 건 규모에서 PostgreSQL의 `SELECT ... ORDER BY created_at LIMIT 3`은 충분하다. 트래픽이 10배 넘게 늘면 그때 전용 큐를 도입한다.

다음 글은 멀티 캐릭터 시스템과 Claude Vision 이미지 분석을 다룬다.
