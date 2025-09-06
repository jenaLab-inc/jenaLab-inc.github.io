---
title: "Armo 웹사이트 개발기 0단계: 프로젝트 부트스트랩 실습 (with pnpm 오류 해결)"
description: "pnpm과 Turborepo로 모노레포를 세팅하면서 발생한 ERR_PNPM_FETCH_404 오류 해결까지 정리한 Armo 웹사이트 개발기 0단계 실습편"
author: "Jerome Na"
date: 2025-09-06
tags: ["Armo 웹사이트", "pnpm workspaces", "turborepo", "모노레포", "yarn vs pnpm", "pnpm 오류 해결"]
keywords: ["pnpm workspaces 예제", "pnpm ERR_PNPM_FETCH_404 해결", "yarn vs pnpm 차이", "turborepo 빌드 최적화", "모노레포 초기 세팅", "Armo 프로젝트"]
---

지난 글([0단계 개요](https://jenalab-inc.github.io/2025/09/05/armo-project-bootstrap.html))에서는 Armo 웹사이트 프로젝트의 개발 규칙과 환경을 기술했다.

이번 글은 그 **실습편**으로, 실제로 모노레포 구조를 만들고 `pnpm`과 `turbo`를 적용하면서 발생한 **오류 해결기**까지 정리하였다.

---

## 1. 모노레포 폴더 구조 만들기

Armo 웹사이트는 `web`, `api`, `admin`, `shared` 네 영역으로 나눴습니다.  
- `web`: Next.js 기반 프론트엔드  
- `api`: Fastify/NestJS 기반 백엔드  
- `admin`: 관리자 대시보드  
- `shared`: 공통 타입과 유틸리티  

```bash
armo/
├─ web/        # Next.js 프론트엔드
├─ api/        # Fastify/NestJS 백엔드
├─ admin/      # 관리자 대시보드
├─ shared/     # 공통 라이브러리
├─ turbo.json
├─ pnpm-workspace.yaml
└─ package.json
```
---

## 2. pnpm vs yarn: 왜 pnpm을 선택했나?

| 비교 항목      | pnpm | yarn |
|----------------|------|------|
| 설치 속도      | 하드링크/심볼릭 링크 활용 → **더 빠름** | 캐시 기반, 상대적으로 느림 |
| 디스크 사용    | 중복 없는 저장 방식 → **공간 절약** | 프로젝트별 중복 설치 가능 |
| workspace 관리 | 설정 단순, 자동 링크 | berry 이후 복잡해짐 |
| node_modules   | 링크 기반 구조 | 전통적 구조 |

Armo 프로젝트는 **효율성과 확장성** 때문에 `pnpm`을 선택했다.

---

## 3. Turborepo: 빌드 최적화 도구

모노레포의 빌드 속도를 책임지는 도구가 **Turborepo**입니다.  

- **캐시 기반 빌드**: 동일한 입력이면 결과를 재사용  
- **병렬 실행**: 독립 패키지 동시 실행  
- **변경 감지**: 수정된 부분만 빌드  

📌 `turbo.json` 예시:

```json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {}
  }
}
```

---
## 4. pnpm Workspaces 설정

📌 루트 pnpm-workspace.yaml

```yaml
packages:
  - "web"
  - "api"
  - "admin"
  - "shared"
```

📌 루트 package.json

```json
{
  "name": "armo",
  "private": true,
  "devDependencies": {
    "turbo": "latest"
  },
  "scripts": {
    "dev": "turbo run dev --parallel",
    "build": "turbo run build",
    "lint": "turbo run lint"
  }
}
```

📌 shared/package.json

```json
{
  "name": "@armo/shared",
  "version": "1.0.0",
  "description": "Shared types and utilities for Armo",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "clean": "rm -rf dist"
  },
  "devDependencies": {
    "@types/node": "^20.14.10",
    "typescript": "^5.5.3"
  },
  "dependencies": {
    "zod": "^3.23.8"
  }
}
```

---
## 5. 내부 패키지 설치 (오류와 해결)

처음에 web 폴더에서 이렇게 실행했다:

```bash
cd web
pnpm add @armo/shared
```

다음과 같은 오류가 발생했다다:

```bash
ERR_PNPM_FETCH_404
```

이 에러는 pnpm이 내부 워크스페이스(shared)를 찾지 못하고 NPM 레지스트리에서 패키지를 찾으려다 실패했기 때문에 생긴다.

---
### 해결 방법

1) pnpm-workspace.yaml 확인

경로가 shared/* 가 아니라 shared로 정확히 잡혀 있어야 한다.

```yaml
packages:
  - "web"
  - "api"
  - "admin"
  - "shared"
```

2) --workspace 옵션 사용

web 폴더에서 실행할 때 이렇게 입력한다:

```bash
pnpm add @armo/shared --workspace
```

3) 설치 후 확인

루트에서:

```bash
pnpm install
```

그리고 web/package.json에는 이렇게 기록된다:

```json
"dependencies": {
  "@armo/shared": "workspace:*"
}
```

---
## 6. 최종 확인

- pnpm dev 로 web/api 동시에 실행 가능

![실행 이미지]({{ site.url }}/assets/article_images/2025-09-06-armo-project-bootstrap2/turbo_screenshot.png)


- shared 패키지가 symlink로 연결되어 코드에서 바로 사용 가능

```ts
// web/src/example.ts
import { type, util } from "@armo/shared";
```

---
## 7. DoD (Definition of Done)
- pnpm dev로 web/api 동시에 실행 OK
- shared 패키지 로컬 연결 확인 OK
- PR 시 Lint/Build 자동 검증 (GitHub Actions) OK

---
## 마무리

이번 실습에서는 pnpm + turborepo 기반 모노레포 초기 세팅을 마치고,
실제로 흔히 마주치는 ERR_PNPM_FETCH_404 오류 해결법까지 다뤘다.