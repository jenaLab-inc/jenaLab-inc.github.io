---
layout: post
ref: storida-system-architecture
title: "Storida 시스템 아키텍처 — AI SaaS를 위한 모노레포 설계"
date: 2026-02-03 10:00:00
categories: architecture
author: jenalab
image: /assets/article_images/storida/architecture.svg
image2: /assets/article_images/storida/architecture.svg
---

## 프로젝트 배경

주인공 이름과 상황을 넣으면 동화책이 나온다. Claude가 글을 쓰고, Gemini가 그림을 그린다. Storida는 그 과정을 자동화한 AI SaaS다.

이 글은 Storida의 시스템 아키텍처와 설계 의사결정을 다룬다.

## 왜 모노레포인가

Storida는 4개의 독립 앱으로 구성된다.

| 앱 | 역할 | 기술 | 포트 |
|---|---|---|---|
| **Web** | 사용자 웹앱 | Next.js + React | 3000 |
| **Admin** | 관리자 패널 | Next.js + React | 3001 |
| **Backend** | API 서버 | Express.js + TypeScript | 4000 |
| **Scheduler** | 배치 처리기 | Node.js + TypeScript | — |

네 앱은 DB 스키마, 타입, 비즈니스 로직을 공유한다. 별도 레포로 분리하면 타입 동기화가 깨지고, 스키마 변경마다 배포 순서를 신경 써야 한다. Yarn Workspaces 모노레포를 선택했다.

```
project/
├── apps/
│   ├── web/          # @myapp/web
│   ├── admin/        # @myapp/admin
│   ├── backend/      # @myapp/backend
│   └── scheduler/    # @myapp/scheduler
├── packages/
│   └── types/        # @myapp/types (공유 타입)
└── package.json      # Yarn Workspaces 루트
```

`packages/types`에 정의한 공유 타입을 4개 앱에서 `@myapp/types`로 import한다. 스키마 변경 시 한 곳만 수정하면 빌드 타임에 불일치가 즉시 감지된다.

## 아키텍처 레이어

4개 레이어로 나뉜다.

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
│   Claude API (텍스트)  ·  Gemini API (이미지)     │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│              Infrastructure Layer                │
│  PostgreSQL (Supabase)  ·  Cloudflare R2         │
│  Firebase Auth  ·  Supabase Realtime             │
└─────────────────────────────────────────────────┘
```

### Presentation → Application

Web과 Admin은 Backend API를 호출한다. 인증은 Firebase ID Token을 `Authorization: Bearer` 헤더로 전달하고, Backend의 미들웨어가 `adminAuth.verifyIdToken()`으로 검증한다.

### Application → Processing

Backend는 AI 생성을 직접 처리하지 않는다. `jobs` 테이블에 Job을 등록하고 즉시 응답한다. Scheduler가 30초 간격으로 PENDING Job을 폴링한다. Vercel은 60초 타임아웃이 있고, AI 생성은 2~4분이 걸린다. 동기 처리는 불가능하다.

### Processing → Infrastructure

Scheduler가 Claude/Gemini API를 직접 호출하고, 결과를 PostgreSQL과 Cloudflare R2에 저장한다. Supabase Realtime으로 Web에 진행률을 실시간 전달한다.

## Feature-Based 디렉토리 구조

모든 앱에서 Feature-Based 구조를 채택했다. 전통적인 레이어별 분리(controllers, services, models) 대신, 도메인 단위로 파일을 묶는다.

```
src/modules/
├── contents/
│   ├── content.routes.ts     # Express 라우트
│   ├── content.service.ts    # 비즈니스 로직
│   └── content.validation.ts # Zod 스키마
├── assets/
├── generate/
├── styles/
├── subscriptions/
└── ...
```

기능을 추가할 때 관련 파일이 한 폴더에 모인다. `subscriptions`를 작업할 때 다른 폴더를 뒤질 필요가 없다. 컨텍스트 전환이 줄어든다.

## 기술 스택 선정 근거

| 영역 | 선택 | 대안 | 선정 이유 |
|---|---|---|---|
| ORM | Drizzle | Prisma | SQL-first 설계, 런타임 번들 경량, Supabase 호환성 |
| Validation | Zod | Joi, Yup | TypeScript 타입 추론, 런타임과 타입 동시 보장 |
| UI | shadcn/ui | MUI, Ant Design | 복사-붙여넣기 방식으로 완전한 커스터마이징, Tailwind 네이티브 |
| 이미지 AI | Gemini Flash | DALL-E, SD | 비용 대비 품질, 한국어 이해도, 캐릭터 일관성 |
| 텍스트 AI | Claude Sonnet | GPT-4 | 한국어 창작 품질, 구조화된 출력 안정성 |

Drizzle ORM은 SQL을 직접 쓰는 감각에 가깝다. 타입 안전성은 유지된다. Prisma의 자동 마이그레이션보다 Drizzle `db:push` + Supabase `db diff` 조합이 운영에서 더 예측 가능했다.

## 배포 구성

```
CloudFlare (CDN + DNS)
    │
    ├── Vercel
    │   ├── Web (Next.js)      ← app.example.com
    │   └── Admin (Next.js)    ← admin.example.com
    │
    ├── Railway
    │   ├── Backend (Express)  ← api.example.com
    │   └── Scheduler (Node)   ← 외부 노출 없음
    │
    ├── Supabase
    │   ├── PostgreSQL          ← DB
    │   └── Realtime            ← 실시간 이벤트
    │
    └── Cloudflare R2
        └── cdn.example.com  ← 이미지 스토리지
```

Vercel은 프론트엔드, Railway는 백엔드와 스케줄러를 호스팅한다. Scheduler는 외부 엔드포인트가 없다. DB를 직접 폴링한다. 각 서비스를 독립 배포하고 수평 확장할 수 있다.

## 핵심 설계 결정 요약

1. **모노레포**: 타입 공유와 스키마 일관성을 위해 Yarn Workspaces 채택
2. **비동기 Job Queue**: Vercel 타임아웃 우회와 동시 처리 제어를 위한 DB 기반 큐
3. **Scheduler 직접 AI 호출**: Backend 경유 없이 성능 최적화
4. **Feature-Based 구조**: 도메인 중심 파일 구성으로 응집도 향상
5. **Supabase Realtime**: 별도 WebSocket 서버 없이 실시간 진행률 전달

다음 글은 Drizzle ORM과 Supabase를 조합한 DB 워크플로우 설계를 다룬다.
