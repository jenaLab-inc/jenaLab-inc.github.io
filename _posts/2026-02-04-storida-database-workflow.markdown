---
layout: post
ref: storida-database-workflow
title: "Drizzle ORM + Supabase — 운영 안전한 DB 워크플로우 설계"
date: 2026-02-04 10:00:00
categories: database
author: jenalab
image: /assets/article_images/storida/database.svg
image2: /assets/article_images/storida/database.svg
---

## 문제: ORM 마이그레이션의 위험성

운영 중인 DB에 컬럼을 추가한다. 자동 마이그레이션 도구가 예측 못한 SQL을 실행한다. 서비스가 멈춘다.

DB 스키마 변경은 배포에서 가장 긴장되는 순간이다. Storida는 Drizzle ORM의 개발 편의성과 Supabase CLI의 운영 안전성을 조합하여 이 문제를 해결했다.

## 워크플로우 개요

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ schema.ts    │────▶│   dev DB     │────▶│ Migration    │
│ (Drizzle)    │ push│  (Supabase)  │ diff│    .sql      │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │ git push
                                           ┌──────▼───────┐
                                           │  prod DB     │
                                           │ (Supabase)   │
                                           └──────────────┘
```

개발 환경에서는 Drizzle이 스키마를 직접 적용한다. 운영 환경에서는 검증된 SQL 마이그레이션만 실행된다.

## 환경 분리

| 환경 | Git 브랜치 | Supabase 브랜치 | 용도 |
|---|---|---|---|
| 개발 | `dev` | Preview | 스키마 실험, 빠른 반복 |
| 운영 | `main` | Production | 검증된 마이그레이션만 적용 |

Supabase는 Git 브랜치 연동 기능을 제공한다. `dev` 브랜치에 푸시하면 Preview DB에 자동 적용되고, `main`에 머지하면 Production DB에 적용된다.

## 3단계 스키마 변경 프로세스

### Step 1. Drizzle 스키마 정의

모든 스키마는 TypeScript로 작성한다. SQL 대신 TypeScript를 사용함으로써 **타입 추론과 자동완성**이 가능해진다.

```
// 콘텐츠 테이블 — Drizzle ORM으로 정의
// 사용자 FK, 제목, 상태(PENDING/GENERATING/COMPLETED),
// 총 페이지 수, 생성 진행률, 생성일시를 저장한다.
// cascade 삭제로 사용자 탈퇴 시 콘텐츠도 함께 삭제된다.
```

Drizzle의 스키마 정의는 SQL DDL과 1:1로 대응한다. Prisma처럼 독자적 DSL을 사용하지 않으므로, 생성될 SQL을 예측하기 쉽다.

### Step 2. 개발 DB에 직접 적용

```bash
cd packages/backend
npm run db:push   # Drizzle → dev DB 직접 적용
```

`db:push`는 현재 스키마와 DB의 차이를 계산하여 즉시 반영한다. 마이그레이션 파일을 생성하지 않으므로 빠른 반복이 가능하다. **개발 환경에서만 사용**한다.

### Step 3. 마이그레이션 SQL 생성

개발 DB에서 충분히 테스트한 후, Supabase CLI로 마이그레이션 SQL을 추출한다.

```bash
supabase db diff -f add_content_type
# → supabase/migrations/20260319_add_content_type.sql
```

이 SQL 파일이 코드 리뷰의 대상이 된다. 자동 생성된 SQL을 사람이 검토함으로써, 운영 환경에 안전하지 않은 DDL(예: `DROP COLUMN`)이 실행되는 것을 방지한다.

## 12+ 테이블 스키마 설계

12개 이상의 테이블이 있다. 핵심 관계는 다음과 같다.

```
사용자 ─────┬── 콘텐츠 ──── 페이지
            │      └───── 캐릭터 연결
            ├── 구독
            ├── 결제
            ├── 크레딧 이력
            └── 크레딧 구매

프롬프트 템플릿 (작문 스타일)
스타일 (그림풍)
AI 모델 설정
공지사항 ── 읽음 기록
약관
Job Queue
사용량 기록 (토큰 사용 로그)
시스템 설정
```

### ID 타입 결정: UUID vs Text

`users.id`와 `managers.id`는 `text` 타입이다. Firebase UID는 28자 영숫자 문자열이다. 처음에는 `uuid`로 설계했다. Firebase Auth 마이그레이션 때 모든 FK 컬럼을 함께 변경해야 했다. 외부 인증 시스템의 ID 형식에 의존하는 테이블은 유연한 타입을 써야 한다.

## 금지 규칙

실수를 방지하기 위해 팀 규칙으로 다음을 정했다.

| 금지 | 이유 |
|---|---|
| Supabase Dashboard에서 직접 테이블 수정 | Drizzle 스키마와 동기화 깨짐 |
| Production DB에 `db:push` 실행 | 마이그레이션 이력 없이 적용됨 |
| 마이그레이션 파일 수동 수정 | 이미 적용된 마이그레이션과 충돌 |

## 시드 데이터 관리

초기 데이터(요금제, 기본 캐릭터, 시스템 설정 등)는 `src/db/seed.ts`에서 관리한다. `npm run db:seed` 명령으로 실행하며, **멱등성(idempotency)**을 보장하도록 `ON CONFLICT DO NOTHING` 패턴을 사용한다.

```
// 시드 데이터 삽입 — 요금제, 기본 캐릭터, 시스템 설정 등
// ON CONFLICT DO NOTHING 패턴으로 멱등성 보장
// 예: 요금제별 이름, 가격, 월 토큰 할당량을 초기 데이터로 등록
```

## 결론

Drizzle ORM은 SQL-first 설계다. "ORM이 무엇을 하는지 모르겠다"는 불안이 없다. Supabase CLI의 `db diff`와 브랜치 기반 자동 배포를 조합하면, 개발 속도와 운영 안전성을 동시에 확보할 수 있다.

다음 글은 Claude와 Gemini를 위한 동적 프롬프트 시스템 설계를 다룬다.
