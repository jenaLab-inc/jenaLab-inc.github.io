---
title: "Armo 웹사이트 개발기 0단계: 프로젝트 부트스트랩"
date: 2025-09-05
tags: [Armo, 웹사이트 개발, Next.js, Fastify, Supabase, Fly.io, Vercel, Monorepo, 프로젝트 세팅, 아키텍처]
description: "Armo 웹사이트 구축의 첫 단계인 프로젝트 부트스트랩 과정을 공유합니다. Monorepo 구조, 환경변수, CI/CD 자동화까지 초기 세팅을 통한 안정적인 개발 환경 마련하기."
keywords: ["Armo 웹사이트", "프로젝트 부트스트랩", "Next.js Vercel", "Fastify Fly.io", "Supabase Postgres", "Monorepo 구조", "웹 개발 아키텍처", "SEO 최적화 블로그"]
canonical: "https://jenalab.github.io/armo-project-bootstrap"
author: "Jerome Na"
---

Armo 웹사이트는 단순히 상품을 판매하는 곳이 아니라, **작가의 작품을 스토리텔링으로 전시하고 굿즈와 패키지로 연결하는 웹 갤러리 플랫폼**을 목표로 한다.  
이번 글에서는 그 시작점인 **프로젝트 부트스트랩(bootstrap)** 과정을 정리한다.

---

## 🚀 왜 부트스트랩이 중요한가?

많은 프로젝트가 코드 작성보다 먼저 **환경 세팅**에서 무너진다.

- 팀마다 환경이 달라 빌드 에러가 반복되거나,  
- 환경 변수가 정리되지 않아 배포 때마다 장애가 생긴다.  

Armo는 현재 최소의 인력으로 개발을 시작하지만, **벤더 종속성을 최소화한 독립 아키텍처**를 지향한다. 따라서 초기부터 체계적인 세팅이 필수다.

---

## 📂 1. Monorepo 구조

Armo 프로젝트는 **Monorepo(모노레포)** 방식으로 진행한다.  
👉 여러 프로젝트(프론트, 백엔드, 어드민)를 하나의 레포지토리에서 관리하는 방식이다.

```ini
armo/
├─ web/     # Next.js 프론트엔드 (Vercel 배포)
├─ api/     # Fastify 백엔드 API (Fly.io 배포)
├─ admin/   # 관리자 페이지
└─ packages/# 공통 라이브러리 (UI, 타입 정의)
```

✅ 장점:
- 공통 코드(컴포넌트/타입)를 쉽게 공유  
- 한 번의 PR에서 여러 서비스 수정 가능  
- CI/CD 파이프라인 일괄 관리  

---

## ⚙️ 2. 공통 개발 규칙 세팅

- **Lint & Formatter**: ESLint + Prettier  
- **Commit 규칙**: Conventional Commit + Husky hook  
- **EditorConfig**: 코드 스타일 통일  

예시 `.editorconfig`:

```ini
root = true

[*]
charset = utf-8
indent_style = space
indent_size = 2
end_of_line = lf
insert_final_newline = true
```
---
## 🔑 3. 환경변수 정의

아키텍처에서 필요한 최소 환경변수:

```bash
# Database
DATABASE_URL=postgres://...

# Storage
STORAGE_BUCKET=armo-assets
STORAGE_REGION=ap-northeast-2

# Auth
JWT_SECRET=super_secret_value

# API
API_BASE_URL=https://api....
```

- 보안 키는 .env에만 저장하고 Git에는 절대 커밋하지 않는다.
- Vercel / Fly.io 각각 환경변수로 주입한다.

---

## 🔄 4. CI/CD 자동화

GitHub Actions로 PR → Build/Lint → 배포 파이프라인 구축:

```yaml
name: CI

on:
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
        with:
          version: 8
      - run: pnpm install
      - run: pnpm lint
      - run: pnpm build
```

이렇게 하면 PR 올릴 때마다 자동 검증이 돌아가므로, 품질 유지가 가능하다.

---

## ✅ DoD (Definition of Done)

- pnpm dev로 web과 api 동시 실행 확인
- .editorconfig, ESLint, Prettier, Husky 정상 작동 확인
- GitHub Actions 빌드 + Lint 성공 확인

---

✨ 마치며

Armo 프로젝트의 첫 단계, 프로젝트 부트스트랩은 단순한 세팅이 아니다.

**앞으로의 개발과 운영을 안전하게 이어가기 위한 기반**이다.