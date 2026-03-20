---
layout: post
ref: storida-character-system
title: "멀티 캐릭터 시스템 — Claude Vision으로 이미지 일관성 확보하기"
date: 2026-02-18 10:00:00
categories: ai
author: jenalab
image: /assets/article_images/storida/character-system.svg
image2: /assets/article_images/storida/character-system.svg
---

## 문제: 페이지마다 달라지는 캐릭터

1페이지에서 갈색 머리의 소녀가 3페이지에서 금발이 된다. 5페이지에서는 완전히 다른 사람이다. AI 이미지 생성의 고질적 문제다.

기존 시스템은 캐릭터 이미지를 "참조 이미지"로 전달했다. Gemini가 이미지에서 무엇을 봐야 하는지 가이드가 없었다.

## 해결 전략: 이미지 분석 → 텍스트 설명 → 프롬프트 주입

```
관리자 이미지 업로드 (최대 5장)
         │
         ▼
Claude Vision 분석 ──────▶ 구조화된 설명 (영문)
                           │
                           ▼
이미지 생성 시 프롬프트에 주입 ──▶ Gemini 이미지 생성
```

이미지를 텍스트로 변환한다. Claude Vision이 캐릭터 이미지를 분석하여 영문 설명을 생성한다. 이 설명을 Gemini 이미지 프롬프트에 주입한다.

## DB 스키마 변경

### characters 테이블

캐릭터 테이블에 분석 결과(한국어/영문 설명)를 JSONB로 저장하는 컬럼을 추가했다.

### content_assets 테이블

콘텐츠-캐릭터 연결 테이블에 커스텀 이름과 주인공 여부 컬럼을 추가했다. 이 테이블이 콘텐츠와 캐릭터의 N:M 관계를 관리하며, 주인공/보조 캐릭터를 구분한다.

## Claude Vision 분석 API

```
POST /api/assets/analyze
// 1~5장의 이미지 URL을 전달
// 한국어/영문 캐릭터 외형 설명을 반환
```

### 분석 프롬프트

Claude에게 캐릭터 이미지를 전달한다. 이미지 생성 AI가 일관되게 재현할 수 있는 시각적 특징을 추출하도록 지시한다.

분석 결과 예시:

```json
{
  "description_ko": "갈색 곱슬머리의 소녀, 큰 갈색 눈,
    둥근 볼, 분홍색 원피스, 운동화 착용",
  "description_en": "A young girl with curly brown hair,
    large brown eyes, round cheeks, wearing a pink dress
    and sneakers, approximately 6-7 years old"
}
```

한국어 설명은 관리자가 확인·편집할 수 있고, 영문 설명은 Gemini 프롬프트에 직접 사용된다.

## 멀티 캐릭터 생성 요청

### 변경 전

단일 캐릭터 ID만 전달하는 방식이었다.

### 변경 후

최대 3개 캐릭터를 전달하며, 각 캐릭터에 커스텀 이름과 주인공 여부를 지정한다. Zod validation으로 **정확히 1명의 주인공**이 포함되어야 함을 검증한다. 1~3개 캐릭터 범위도 함께 검증한다.

## Scheduler: 프롬프트에 캐릭터 정보 주입

### 텍스트 생성 (Claude)

{% raw %}
작문 프롬프트 템플릿에 캐릭터 정보 변수를 추가했다.

```
### 주인공 캐릭터
- 이름: {{주인공_이름}}
- 종류: {{캐릭터_종류}}
- 외형: {{캐릭터_외형_설명}}

### 보조 캐릭터
{{보조_캐릭터_정보}}
```

`{{캐릭터_외형_설명}}`에 `analysis_report.description_ko`가, 보조 캐릭터 정보에 각 보조 캐릭터의 이름과 설명이 들어간다.
{% endraw %}

### 이미지 생성 (Gemini)

```
Main character: A young girl with curly brown hair,
  large brown eyes, round cheeks, wearing a pink dress
  and sneakers. Her name is "토토".

Supporting character: A small white rabbit with long ears
  and a blue ribbon. Name: "미미".

[참조 이미지 첨부]

Style: watercolor illustration, soft and gentle...

Scene: 토토와 미미가 숲 속에서 나비를 쫓아갔어요.
```

`analysis_report.description_en`이 텍스트로 주입된다. 이미지 AI가 매 페이지에서 동일한 시각적 특징을 재현한다. 참조 이미지만 전달할 때와 차이가 뚜렷하다.

## Web UI: 멀티 선택 인터페이스

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│ ★ 토토  │  │ ✓ 미미  │  │ ✓ 뭉치  │
│  (주인공) │  │  (보조)  │  │  (보조)  │
│ 노란 ring│  │ 파란 ring│  │ 파란 ring│
└─────────┘  └─────────┘  └─────────┘
```

- 첫 번째 선택 = 자동으로 주인공(★ + 노란색 ring)
- 추가 선택(최대 2개) = 보조 캐릭터(✓ + 파란색 ring)
- 토글 방식: 클릭으로 선택/해제

## 이미지 분석의 한계와 대응

Claude Vision 분석에는 한계가 있다.

1. **추상적 캐릭터**: 캐릭터가 실제 사물이 아닌 판타지 존재일 때 설명이 모호해짐
2. **다중 이미지 불일치**: 5장의 이미지가 서로 다른 앵글이나 스타일이면 통합 설명이 어려움
3. **색상 정확도**: 미묘한 색상 차이를 텍스트로 정확히 전달하기 어려움

분석 결과를 관리자가 편집할 수 있도록 Textarea를 제공한다. AI 분석은 초안이다. 관리자가 최종 검수한다.

## 성과

| 지표 | 개선 전 | 개선 후 |
|---|---|---|
| 캐릭터 일관성 | 페이지별 외모 변동 빈번 | 핵심 특징(머리색, 의상) 유지 |
| 캐릭터 수 | 1명만 가능 | 최대 3명 |
| 프롬프트 품질 | 참조 이미지만 전달 | 텍스트 설명 + 이미지 병행 |
| 관리자 작업 | 수동 특징 입력 | AI 분석 + 편집 |

다음 글에서는 토큰 기반 요금제 시스템 설계를 다룬다.
