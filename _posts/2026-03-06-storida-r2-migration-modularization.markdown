---
layout: post
ref: storida-r2-migration-modularization
title: "Cloudflare R2 마이그레이션과 Scheduler 모듈화 리팩토링"
date: 2026-03-06 10:00:00
categories: infrastructure
author: jenalab
image: /assets/article_images/storida/r2-migration.svg
image2: /assets/article_images/storida/r2-migration.svg
---

## Part 1: Supabase Storage에서 Cloudflare R2로

### 마이그레이션 동기

Supabase Storage는 초기 개발에 편리했다. 서비스가 성장하면서 한계가 드러났다.

| 기준 | Supabase Storage | Cloudflare R2 |
|---|---|---|
| 비용 | 무료 티어 한정, 이후 급증 | 10GB 무료, 이후 $0.015/GB |
| Egress 비용 | 있음 | **무료** |
| CDN | Supabase CDN | Cloudflare 글로벌 CDN 자동 |
| 커스텀 도메인 | 제한적 | 자유 설정 |
| Presigned URL | 미지원 | S3 호환 API 완전 지원 |

Egress 비용이 결정적이었다. 동화책 서비스는 이미지 트래픽이 많다. 페이지당 이미지가 있고, 사용자가 반복 열람한다. R2는 Egress가 무료다.

### R2 구성

| 환경 | 버킷 | 공개 URL |
|---|---|---|
| 개발 | `myapp-dev` | `cdn-dev.example.com` |
| 운영 | `myapp` | `cdn.example.com` |

디렉토리 구조를 Supabase 버킷과 1:1 매핑하여 마이그레이션의 복잡도를 최소화했다.

| 기존 Supabase 버킷 | R2 디렉토리 | 용도 |
|---|---|---|
| `media` | `media/` | 장면 이미지, PDF, 미리보기 |
| `shared` | `shared/` | 폰트, 공지 이미지 |
| (신규) | `assets/` | 캐릭터 이미지 |

### Presigned URL 워크플로우

기존에는 클라이언트가 Supabase SDK로 직접 업로드했다. R2에서는 Presigned URL 패턴으로 전환했다.

```
Client → POST /api/upload/presign
         { bucket: "assets", path: "abc.jpg", contentType: "image/jpeg" }
       ← { uploadUrl: "https://...(서명된 URL)", publicUrl: "https://cdn.example.com/assets/abc.jpg" }

Client → PUT uploadUrl (파일 바이너리 전송)

Client → publicUrl로 접근
```

이 패턴의 장점:

1. **Backend가 파일을 중계하지 않음**: 대역폭 절약
2. **권한 검증을 Backend에서 수행**: Presign 요청 시 인증 확인
3. **클라이언트 SDK 불필요**: 표준 HTTP PUT으로 업로드

### 레거시 URL 호환 전략

기존 DB에는 Supabase Storage URL이 저장되어 있다. 한꺼번에 바꾸면 위험하다. 점진적 전환을 택했다.

- 기존 Supabase URL: 유지 (Supabase Storage public 설정 유지)
- 새 업로드: R2 URL로 저장
- `deleteFile`: R2 URL만 삭제, Supabase URL은 warn 로그만 출력

시간이 지나면서 자연스럽게 R2 URL 비율이 높아지고, 충분히 전환된 후 Supabase Storage를 제거한다.

### 변경 범위

| 앱 | 변경 사항 |
|---|---|
| Backend | `supabase-storage.ts` → `r2-storage.ts` (동일 인터페이스) |
| Scheduler | Supabase SDK 제거, `uploadToR2()` 함수 추가 |
| Admin | Supabase 직접 업로드 → Presigned URL 방식 |
| Web | PDF 생성 시 Presigned URL 사용 |

`r2-storage.ts`는 기존 `supabase-storage.ts`와 동일한 함수 시그니처를 유지한다. `uploadImage`, `downloadImage`, `deleteFile` — import 경로만 바꾸면 된다.

---

## Part 2: Scheduler 모듈화 리팩토링

### 문제: 638줄 단일 파일

R2 마이그레이션을 진행하면서 Scheduler의 `src/index.ts`가 638줄이라는 사실을 직면했다. DB 스키마, Lock 관리, 토스페이먼츠 API, 자동 결제, 공지 발행, Job 복구, Job 처리 루프가 모두 하나의 파일에 있었다.

### 모듈화 원칙

**역할 단위 분리**와 **의존성 주입** 두 가지 원칙으로 리팩토링했다.

### 변경 전

```
scheduler/src/
├── constants.ts
├── index.ts              # 638줄 — 모든 로직
└── services/
    └── generation.service.ts
```

### 변경 후

```
scheduler/src/
├── constants.ts
├── index.ts              # ~100줄 — 메인 루프만
├── db/
│   ├── schema.ts         # Drizzle 테이블 정의
│   └── client.ts         # DB 연결 팩토리
├── utils/
│   └── lock.ts           # Lock 파일 관리
└── services/
    ├── payment-gateway.ts    # 결제 API 유틸
    ├── publisher.service.ts  # 예약 발행
    ├── billing.service.ts    # 자동 결제 (3개 함수)
    ├── recovery.service.ts   # Stale Job 복구
    ├── job.service.ts        # Job 처리 루프
    └── generation.service.ts # AI 생성 파이프라인
```

### DB 클라이언트 팩토리 패턴

모듈 스코프 변수를 캡처하는 대신, **팩토리 함수**로 DB 클라이언트를 생성하고 각 서비스에 주입한다.

DB 클라이언트 팩토리 함수가 연결을 생성하고, 메인 엔트리에서 이를 각 서비스에 주입한다. 모든 서비스 함수가 DB 클라이언트를 첫 번째 파라미터로 받는다. 테스트 시 Mock DB를 주입할 수 있고, 함수가 **외부 상태에 의존하지 않는** 순수한 구조가 된다.

### 토스 API 시그니처 변경

기존에 모듈 스코프 변수를 클로저로 캡처하던 방식을 명시적 파라미터로 전환했다.

기존에는 모듈 스코프 변수를 클로저로 캡처하여 암묵적으로 사용했다. 리팩토링 후 모든 외부 의존성을 명시적 파라미터로 전달한다. 결제 API 키, DB 클라이언트 등이 함수 시그니처에 드러난다.

### index.ts의 역할

리팩토링 후 `index.ts`는 **오케스트레이터** 역할만 한다.

```
index.ts 담당:
1. 환경변수 검증 (DATABASE_URL 필수)
2. Lock 획득 (중복 실행 방지)
3. Signal 핸들러 등록 (SIGTERM, SIGINT)
4. DB 클라이언트 생성
5. 3개 타이머 등록:
   - 30초: Job 처리
   - 60초: 공지 발행 + Job 복구
   - 1시간: 자동 결제
```

### 무변경 보장

동작 변경은 없다. 로직 자체는 그대로이며 파일 구조만 변경했다. `tsup`이 단일 번들로 컴파일하므로 배포에도 영향 없다. `yarn build:scheduler`로 빌드 검증 후 배포했다.

## 두 작업의 연관성

R2 마이그레이션과 모듈화를 같은 날 진행했다. 638줄 파일에서 Storage 코드를 찾아 교체하려니 비효율적이었다. 모듈을 먼저 분리하니 변경 범위가 명확해졌다.

리팩토링은 미루면 영원히 하지 않는다. 필요한 시점에 필요한 만큼 하는 편이 낫다.

다음 글은 콘텐츠 타입 확장 설계를 다룬다.
