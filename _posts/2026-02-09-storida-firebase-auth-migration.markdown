---
layout: post
ref: storida-firebase-auth-migration
title: "인증 시스템 마이그레이션 — Supabase Auth에서 Firebase Auth로"
date: 2026-02-09 09:00:00
categories: auth
author: jenalab
image: /assets/article_images/storida/auth-migration.svg
image2: /assets/article_images/storida/auth-migration.svg
---

## 마이그레이션 배경

초기 인증은 Supabase Auth를 썼다. DB, Storage, Auth를 한 플랫폼에서 관리했다. 그러다 JenaLab의 다른 서비스(Armo, Contexta)와 통합 인증이 필요해졌다. Supabase Auth의 한계가 드러났다.

Firebase Auth(`shared-auth`)를 선택한 이유:

| 기준 | Supabase Auth | Firebase Auth |
|---|---|---|
| 멀티 프로젝트 인증 | 프로젝트별 독립 | 단일 프로젝트로 통합 가능 |
| Google OAuth | 지원 | 네이티브 지원 + 간편 설정 |
| Admin SDK | 제한적 | 사용자 CRUD, 커스텀 클레임 완전 지원 |
| 모바일 SDK | 제한적 | Flutter/Swift/Kotlin 네이티브 |

JenaLab의 모든 서비스가 `shared-auth` 하나를 공유한다. 한 번 가입하면 모든 서비스를 쓸 수 있다.

## 마이그레이션 전략: 6단계 Phase

전체 교체를 한 번에 진행하지 않고, 6단계로 나누어 단계별로 빌드 검증을 수행했다.

```
Phase 1  환경변수 준비
Phase 2  의존성 설치
Phase 3  DB 스키마 변경     ← 가장 위험한 단계
Phase 4  Backend 인증 교체
Phase 5  Web 인증 교체
Phase 6  Admin 인증 교체
```

### Phase 3: DB 스키마 변경 — 가장 위험한 단계

`users.id`의 타입이 바뀐다. Supabase Auth는 UUID다. Firebase Auth는 28자 영숫자 문자열이다.

```
-- 사용자 ID 타입을 uuid에서 text로 변경
-- 사용자를 참조하는 모든 FK 컬럼(9개 이상)도 함께 타입 변경
-- FK 제약을 일시 해제 후 재생성하는 과정이 필요
```

이 변경의 위험성:

1. **FK 제약 일시 해제 필요**: 타입 변경 시 FK 제약을 DROP 후 재생성
2. **기존 데이터 호환**: 기존 UUID 데이터는 text로 자동 변환되지만, 새 Firebase UID와 형식이 다름
3. **롤백 복잡도**: text → uuid 역변환은 데이터 손실 위험

이 단계는 **개발 DB에서 먼저 실행**하고, 마이그레이션 SQL을 생성하여 코드 리뷰 후 운영에 적용했다.

### Phase 4: Backend 인증 미들웨어 교체

```
// 변경 전: Supabase Auth — 토큰으로 사용자 조회 (네트워크 호출)
// 변경 후: Firebase Admin — JWT 로컬 검증 (네트워크 호출 없음)
```

`verifyIdToken`은 JWT를 검증하고 디코딩한다. 별도 네트워크 호출이 없다. 인증 레이턴시가 줄었다.

### Phase 5: Web 클라이언트 인증

Web 앱은 Firebase의 클라이언트 SDK를 사용한다.

```
// Firebase 클라이언트 SDK 인증 흐름
// 1. 이메일/비밀번호 또는 Google OAuth로 로그인
// 2. 인증 상태 변경 감지 (onAuthStateChanged)
// 3. Firebase ID Token을 획득하여 API 요청에 사용
```

Supabase Auth는 쿠키 기반 세션이었다. Firebase는 클라이언트 사이드 토큰 관리 방식이다. `Authorization: Bearer {token}` 헤더로 Backend에 전달한다.

## Supabase의 새로운 역할

인증을 Firebase로 이전한 후, Supabase의 역할이 명확해졌다.

```
┌─────────────────────────────────────────┐
│              Firebase                    │
│  Authentication (shared-auth)        │
│  ID Token 발급 · 사용자 관리             │
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│              Supabase                    │
│  PostgreSQL  · Realtime · (Storage*)    │
│  *Storage는 이후 R2로 마이그레이션       │
└─────────────────────────────────────────┘
```

각 플랫폼이 가장 잘하는 영역만 담당하게 되었다.

## Firebase 에러 메시지 한국어 처리

사용자 경험을 위해 Firebase 에러 코드를 한국어로 번역하는 함수를 구현했다.

```
// Firebase 에러 코드 → 한국어 메시지 매핑
// 예: 'auth/email-already-in-use' → '이미 등록된 이메일입니다.'
// 예: 'auth/wrong-password' → '비밀번호가 올바르지 않습니다.'
// 매핑되지 않은 코드는 기본 에러 메시지를 반환
```

## 교훈

1. **ID 타입은 유연하게**: 외부 인증 시스템을 사용할 때 `text` 타입이 `uuid`보다 안전하다
2. **Phase별 빌드 검증**: 대규모 마이그레이션은 반드시 단계별로 빌드를 확인한다
3. **역할 분리**: 하나의 플랫폼에 모든 것을 의존하면 마이그레이션 비용이 기하급수적으로 증가한다

다음 글에서는 같은 날 구현한 토스페이먼츠 구독 결제 시스템을 다룬다.
