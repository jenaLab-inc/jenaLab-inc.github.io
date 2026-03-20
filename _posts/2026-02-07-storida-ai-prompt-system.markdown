---
layout: post
ref: storida-ai-prompt-system
title: "AI 프롬프트 엔지니어링 — 동적 프롬프트 시스템과 AI 분석 파이프라인"
date: 2026-02-07 10:00:00
categories: ai
author: jenalab
image: /assets/article_images/storida/prompt-system.svg
image2: /assets/article_images/storida/prompt-system.svg
---

## 문제: 하드코딩된 프롬프트

AI SaaS에서 가장 자주 바뀌는 건 프롬프트다. 톤을 조정한다. 제약 조건을 고친다. 새 스타일을 추가한다. 매번 코드 배포가 필요했다.

운영팀이 프롬프트를 즉시 수정할 수 있어야 했다. 템플릿 변수 치환 시스템과 AI 기반 자동 분석 파이프라인으로 이 문제를 풀었다.

## 프롬프트 아키텍처

프롬프트는 두 유형으로 나뉜다.

| 유형 | AI 모델 | 관리 위치 | 용도 |
|---|---|---|---|
| 작문 프롬프트 | Claude Sonnet 4 | Admin > 시스템 설정 + 작문 스타일 | 동화 텍스트 생성 |
| 이미지 프롬프트 | Gemini 2.0 Flash | Admin > 그림풍 관리 | 삽화 이미지 생성 |

## 템플릿 변수 치환 시스템

### 설계 원리

{% raw %}
프롬프트 템플릿은 Admin의 시스템 설정에서 마크다운 형태로 관리된다. `{{변수명}}` 형식의 플레이스홀더를 사용하며, 생성 시점에 실제 값으로 치환된다.

```
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│ 템플릿 (DB)   │────▶│ 변수 수집     │────▶│ 최종 프롬프트  │
│ {{주인공_이름}}│     │ 사용자 입력   │     │ "토토"         │
│ {{작문_프롬프트}}│   │ DB 조회       │     │ 실제 스타일... │
└───────────────┘     └───────────────┘     └───────────────┘
```

### 변수 목록

| 변수 | 출처 | 설명 |
|---|---|---|
| `{{주인공_이름}}` | 사용자 입력 | 동화 주인공 이름 |
| `{{이야기_상황}}` | 사용자 입력 | 이야기 배경 상황 |
| `{{캐릭터_종류}}` | characters 테이블 | 캐릭터 종류 태그 |
| `{{캐릭터_설명}}` | characters 테이블 | 캐릭터 특징 |
| `{{작문_프롬프트}}` | prompt_templates 테이블 | AI가 분석한 스타일 프롬프트 |
| `{{장면_수}}` | 사용자 입력 | 생성할 페이지 수 |
| `{{주인공_최대_글자}}` | plans 테이블 | 요금제별 글자수 제한 |
{% endraw %}

### 치환 함수

{% raw %}치환 함수는 템플릿의 플레이스홀더를 실제 값으로 교체한다. 정규식으로 `{{변수명}}` 패턴을 찾아 대응하는 값을 넣는다.{% endraw %}

단순한 정규식 치환이지만, 핵심은 **모든 변수가 DB에서 조회**된다는 점이다. 코드 배포 없이 Admin에서 템플릿을 수정하면 즉시 반영된다.

## AI 분석 파이프라인: 작문 스타일

글 샘플을 입력하면 AI가 스타일을 자동 분석한다.

### 플로우

```
관리자 입력                     Claude 분석               결과
┌──────────────┐              ┌──────────────┐         ┌──────────────┐
│ 글 샘플 2~3개 │─── POST ───▶│ Claude       │────────▶│ display_name │
│ (한글 텍스트) │  /analyze   │ Sonnet 4     │         │ tone         │
└──────────────┘              └──────────────┘         │ narrative    │
                                                       │ generated    │
                                                       │ _prompt      │
                                                       └──────────────┘
```

관리자가 "이솝우화 느낌의 교훈적인 글"이라고 입력하는 대신, 해당 스타일의 **실제 글 샘플을 2~3개 붙여넣기**한다. Claude가 이를 분석하여 톤, 서술 시점, 문체 특징을 자동 추출하고, 다른 AI 모델이 참고할 수 있는 `generated_prompt`를 생성한다.

{% raw %}이 `generated_prompt`가 템플릿의 `{{작문_프롬프트}}` 변수에 치환된다.{% endraw %}

## AI 분석 파이프라인: 그림풍

그림풍(스타일) 관리는 3단계 워크플로우로 구성된다.

### 3단계 플로우

| 단계 | 입력 | 출력 |
|---|---|---|
| 1. 설명 입력 | 한글 설명 (최소 10자) | — |
| 2. AI 분석 | Claude 자동 처리 | 영문 style_prompt, keywords, negative_keywords |
| 3. 미리보기 | Gemini 자동 처리 | 512x512 샘플 이미지 |

### AI 분석 결과 구조

```json
{
  "name": "수채화 동화풍",
  "style_prompt": "watercolor illustration, soft and gentle,
    pastel colors, children's book style, warm lighting,
    hand-painted texture",
  "keywords": ["watercolor", "soft", "pastel", "warm"],
  "negative_keywords": ["realistic", "dark", "scary"]
}
```

관리자가 "따뜻한 수채화 느낌의 부드러운 그림체"라고 한글로 입력하면, Claude가 영문 이미지 생성 프롬프트로 변환한다. 이 `style_prompt`가 Gemini 이미지 생성 시 직접 사용된다.

### 이미지 프롬프트 조합

최종 이미지 프롬프트는 장면 설명과 스타일 정보를 결합한다.

```
Generate a children's book illustration for page 1 of 5.
The image must be in portrait orientation (3:4 aspect ratio).
Do NOT include any text, letters, words, or numbers.

Style: watercolor illustration, soft and gentle, pastel colors...
Keywords: watercolor, soft, pastel, children, warm
Avoid: realistic, dark, scary

Scene description:
토토는 숲 속에서 길을 잃고 두리번거렸어요.
"엄마, 어디 계세요?"
```

## 요금제별 동적 제한

같은 템플릿을 쓰되, 요금제에 따라 입력 범위가 달라진다.

요금제 정보를 기반으로 Zod 입력 검증 규칙을 런타임에 구성한다. 이름 글자수, 상황 설명 길이, 페이지 수 제한이 플랜에 따라 달라진다. 이 제한값은 프롬프트 변수에도 반영되어, AI가 적절한 분량의 콘텐츠를 생성하도록 가이드한다.

## 전체 플로우 요약

```
사용자 입력 (Web)
    │
    ▼
Backend 검증 ←── plans 테이블에서 동적 제한값 조회
    │
    ▼
Job 등록 (플랜 제한값 포함)
    │
    ▼
Scheduler 처리
    ├── app_config에서 작문 템플릿 조회
    ├── prompt_templates에서 스타일 프롬프트 조회
    ├── 변수 치환 → 최종 프롬프트 생성
    ├── Claude API 호출 → 텍스트 생성
    │
    ├── styles에서 그림풍 정보 조회
    ├── 장면별 이미지 프롬프트 조합
    └── Gemini API 호출 → 이미지 생성
```

## 설계 결정의 트레이드오프

| 결정 | 장점 | 단점 |
|---|---|---|
| DB 기반 프롬프트 | 배포 없이 즉시 변경 | DB 장애 시 폴백 필요 |
| AI 자동 분석 | 비전문가도 스타일 등록 가능 | AI 분석 비용 발생 |
| 변수 치환 방식 | 단순하고 예측 가능 | 조건부 로직 표현 어려움 |

단순성과 표현력 사이의 선택이다. Jinja2나 Handlebars를 쓰면 조건분기가 가능하다. 대신 프롬프트 디버깅이 어려워진다. Storida는 단순 치환을 택했다. 복잡한 분기가 필요하면 별도 템플릿을 등록한다.

다음 글은 Supabase Auth에서 Firebase Auth로의 전환 과정을 다룬다.
