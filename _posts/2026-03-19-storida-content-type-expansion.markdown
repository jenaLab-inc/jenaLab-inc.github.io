---
layout: post
ref: storida-content-type-expansion
title: "콘텐츠 타입 확장 설계 — 동화 전용에서 범용 창작 플랫폼으로"
date: 2026-03-19 10:00:00
categories: architecture
author: jenalab
image: /assets/article_images/storida/content-expansion.svg
image2: /assets/article_images/storida/content-expansion.svg
---

## 문제: 30곳에 하드코딩된 "동화"

"어린이 동화" 전용 서비스로 시작했다. "동화", "아이들", "children's book"이 30곳 이상에 박혀 있었다. 성인 에세이를 생성해도 "어린이가 이해하기 쉬운 단어"가 적용된다. 부적절한 결과가 나왔다.

| 하드코딩 위치 | 내용 |
|---|---|
| scheduler/src/context.ts | "어린이 동화를 JSON 형식으로..." |
| scheduler/src/image-regen.ts | "children's book illustration" |
| src/db/seed.ts | 시스템 프롬프트, 평가 기준 |
| src/modules/generation.service.ts | 폴백 제목 "생성된 동화" |
| src/modules/prompt.routes.ts | "동화 샘플", "어린이 콘텐츠 적합성" |

## 설계 원칙

5가지 원칙으로 확장을 설계했다.

| 원칙 | 적용 |
|---|---|
| **하위 호환 100%** | `content_type` 기본값 `fairytale` → 기존 코드 무변경 시 동일 동작 |
| **기존 패턴 준수** | Feature-based 구조, Drizzle ORM, Express, Zod 그대로 |
| **단일 진실 원천** | 프롬프트는 DB에서 조회, 하드코딩 폴백은 타입별 분기 |
| **점진적 확장** | `content_type`을 text 타입 → 추후 `essay`, `poem` 등 추가 용이 |
| **최소 변경** | 기존 테이블에 컬럼 추가 우선, 새 테이블은 `writings`만 |

## 콘텐츠 타입 정의

| 값 | 라벨 | 대상 | 어휘 제한 |
|---|---|---|---|
| `fairytale` | 동화 | 어린이 | 쉬운 어휘, 교육적 |
| `general` | 일반 | 성인 | 제한 없음, 자유 문체 |

기본값은 `fairytale`이다. 기존 사용자는 아무 변경 없이 동일한 경험을 유지한다.

## 핵심 아키텍처: PromptResolver

### 문제

30곳을 각각 `if (contentType === 'fairytale') ... else ...`로 분기하면 코드가 2배로 늘어난다.

### 해결: 중앙 프롬프트 해석기

```
                    ┌──────────────────┐
 context.ts ───────▶│  PromptResolver  │
 image-regen.ts ───▶│                  │
 generation.svc ───▶│  resolve(key,    │──▶ app_config DB
 prompt.routes ────▶│    contentType)  │──▶ 폴백 상수 맵
                    └──────────────────┘
```

`PromptResolver`는 하나의 진입점으로 모든 프롬프트 조회를 통합한다. `content_type`에 따라 적절한 프롬프트를 반환하되, DB에 없으면 폴백 상수를 사용한다.

PromptResolver는 콘텐츠 타입에 따라 적절한 프롬프트를 반환한다.

```
// 조회 우선순위:
// 1. 캐시 확인
// 2. DB 설정 테이블에서 타입별 키로 조회 (fairytale은 기존 키, general은 접미사 추가)
// 3. 폴백 상수 맵에서 조회
```

### 폴백 상수 맵 설계

폴백 프롬프트는 콘텐츠 타입별로 분리된다. 동화 타입은 아동 친화적 어휘와 교육적 톤을, 일반 타입은 자유로운 문체를 사용한다. 작문 시스템 프롬프트, 이미지 생성 프롬프트, 평가 프롬프트 각각에 타입별 폴백이 존재한다.

DB 설정이 없을 때 폴백이 작동하므로, 새 콘텐츠 타입을 추가할 때 DB 시드를 먼저 넣지 않아도 서비스가 중단되지 않는다.

## DB 스키마 변경

### 기존 테이블 확장

기존 테이블(`contents`, `prompt_templates`, `characters`)에 타입 구분 컬럼을 추가한다. 기본값 설정으로 기존 데이터는 자동 호환된다.

모든 기본값이 기존 동작과 동일하므로, **롤백 시 코드만 되돌리면** DB는 그대로 작동한다.

### 신규 테이블: writings (글쓰기)

글쓰기 테이블은 사용자가 작성한 글 본문, 콘텐츠 타입, 상태(초안/사용됨)를 저장한다. 글쓰기 기능은 "아이디어를 먼저 글로 정리한 뒤, 나중에 책으로 만드는" 2단계 워크플로우를 지원한다. 글에서 생성된 콘텐츠에 대한 참조로 글에서 책으로의 전환을 추적한다.

## 기능 범위 (F1~F10)

| # | 기능 | 설명 |
|---|---|---|
| F1 | 콘텐츠 타입 선택 | 생성 시 "동화"/"일반" 선택 UI |
| F2 | 타입별 프롬프트 분기 | PromptResolver 기반 자동 전환 |
| F3 | DB 스키마 확장 | `contents.content_type` 컬럼 |
| F4 | 하드코딩 제거 | 30곳 → DB 설정 또는 타입 분기 |
| F5 | 글쓰기 기능 | 독립 글 작성·저장 |
| F6 | 글 → 책 전환 | 저장된 글을 "상황" 입력으로 활용 |
| F7 | 자동 캐릭터 프로파일링 | 글 생성 후 등장인물 시각 프로파일 자동 추출 |
| F8 | 캐릭터 이미지 타입 분류 | 인물/풍경/사물 자동 분류 |
| F9 | 작문 프롬프트 타입 연동 | content_type별 프롬프트 분류 |
| F10 | 이미지 모드 | 펼침(16:9) vs 삽화 선택 |

## F7: 자동 캐릭터 프로파일링

캐릭터를 지정하지 않아도 이미지 일관성을 확보하는 기능이다.

```
텍스트 생성 완료
    │
    ▼
Claude가 등장인물 시각 프로파일 자동 추출
(캐릭터별 외형 묘사: 나이, 머리색, 복장 등)
    │
    ▼
이미지 생성 시 프로파일을 프롬프트에 주입
```

기존 멀티 캐릭터 시스템은 관리자가 등록한 캐릭터에 의존했다. F7은 텍스트에서 자동 추출한다. 사용자가 캐릭터를 선택하지 않아도 일관성을 확보한다.

## 변경 영역 매핑

```
packages/db        → schema 확장 (컬럼 추가, writings 테이블)
        ↓
src/modules        → generation 검증, writings CRUD, prompts 타입 분기
        ↓
scheduler/src      → PromptResolver, character-profiler, image 크기 분기
        ↓
web/src            → content_type 선택 UI, 글쓰기 기능
admin/src          → prompts content_type 필드, characters image_type 뱃지
```

## 설계의 핵심: 확장 가능성

`content_type`을 `text` 타입으로 설계했다. 향후 `essay`, `poem`, `novel`을 추가할 때 DB 마이그레이션이 필요 없다. 새 값만 추가하면 된다.

새 타입 추가 시 폴백 프롬프트 추가, DB 설정 시드, UI 옵션 추가만 하면 된다. DB 스키마 변경은 필요 없다.

PromptResolver의 `resolve(baseKey, contentType)` 패턴이 이를 가능하게 한다. 새 타입의 프롬프트가 DB에 없으면 폴백을 사용하고, 점진적으로 DB에 최적화된 프롬프트를 추가하면 된다.

## 결론

"동화 전용"에서 "범용 창작 플랫폼"으로의 확장은 if-else를 추가하는 일이 아니다. 중앙 프롬프트 해석기라는 추상 레이어를 도입하고, 하위 호환 100%를 보장하는 마이그레이션 전략을 세워야 한다.

기존 사용자에게는 아무것도 바뀌지 않는다. 새로운 가능성만 열린다.
