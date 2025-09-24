---
title: "Armo 개발 단계 #1.1 — DoD 달성을 위한 실전 순서"
description: "Prisma와 Postgres(Supabase)로 데이터 모델을 스케치하고 마이그레이션 및 CRUD 검증까지 한 번에 끝내는 체크리스트"
keywords: ["Armo", "데이터 모델", "Prisma", "마이그레이션", "Postgres", "Supabase", "CRUD", "MVP"]
author: "Jerome Na"
date: 2025-09-12
---

이 글은 **1. 데이터 모델 스케치 & 마이그레이션**의 완료 기준(DoD)인  
- `pnpm prisma migrate dev` 성공  
- 로컬 DB에서 **기본 CRUD** 동작을 **순서대로** 달성하는 실전 가이드입니다.  

(목표/테이블/공통필드는 1단계 정의를 따릅니다. 테이블: `artists, artworks, goods, packages,...` / 공통 필드: `id, slug, status, publish_at, created_at, updated_at`)

> 참고: 본 단계는 Armo 1단계(MVP) 중 “데이터 모델 스케치 & 마이그레이션”에 해당합니다. (개발 단계 문서) 

> 예시 소스는 전체 소스가 아닙니다. 구조만 보여주기 위한 소스이기 때문에 ...으로 되어 있는 부분은 각자에 맞게 수정하셔야 합니다.

---

## 0) 사전 준비
- Node.js(LTS), pnpm 설치
- Postgres 인스턴스(로컬 Docker 또는 Supabase Project) 준비  
  - 예) Supabase 생성 후 `DATABASE_URL` 복사

프로젝트 구조(모노레포/분리 모두 OK). 여기서는 `api/` 폴더에서 진행:
```bash
/api
    package.json
    .env
/prisma
    schema.prisma
```

---

## 1) API 프로젝트 초기화
```bash
cd api
pnpm init -y
pnpm add @prisma/client
pnpm add -D prisma
```

package.json 스크립트에 추가:
```json
{
  "scripts": {
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev",
    "seed": "node prisma/seed.cjs",
    "dev": "node index.js"
  }
}
```

## 2) 환경변수 설정 (.env)

.env 파일 생성:
```ini
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DBNAME?schema=public"
```
- Supabase일 경우 Project Settings → Database → Connection string를 그대로 사용.

## 3) Prisma 초기화 & 스키마 작성
```bash
pnpm prisma init
```

prisma/schema.prisma를 아래 예시로 교체(핵심 테이블 + 공통필드 + 관계):
```ts
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Status {
  draft
  scheduled
  published
}

model Artist {
  id        String   @id @default(cuid())
  slug      String   @unique
  ...
}

model Artwork {
  id        String   @id @default(cuid())
  slug      String   @unique
  ...
}

model Goods {
  id        String   @id @default(cuid())
  slug      String   @unique
  ...
}

...
```

* 위 스키마는 1단계에서 정의한 핵심 테이블과 공통 필드, 그리고 작품↔굿즈(다대다), 패키지→구성품(다중 참조) 관계를 반영합니다.

## 4) 마이그레이션 실행
```bash
pnpm prisma generate
pnpm prisma migrate dev --name init
```
* 성공하면 DB에 테이블이 생성됩니다.

## 5) 시드 데이터 삽입 (CRUD 검증용)
prisma/seed.cjs 생성:
```js
// prisma/seed.cjs
const { PrismaClient } = require('@prisma/client')
const prisma = new PrismaClient()

async function main() {
  const artist = await prisma.artist.create({
    data: {
      slug: 'jenalab',
      nameKo: '제나랩',
      tagline: '작품을 일상으로',
      status: 'published'
    }
  })

  ...

  // 작품-굿즈 매핑(M:N)
  await prisma.artworkOnGoods.create({
    data: { artworkId: artwork.id, goodsId: mug.id }
  })

  // 패키지 + 구성품
  const pkg = await prisma.package.create({
    data: {
      slug: 'squirrel-starter-pack',
      nameKo: '다람쥐 스타터 팩',
      status: 'draft'
    }
  })
  await prisma.packageItem.create({
    data: { packageId: pkg.id, goodsId: mug.id, quantity: 1 }
  })

  console.log('Seed OK')
}

main()
  .then(() => prisma.$disconnect())
  .catch((e) => { console.error(e); prisma.$disconnect(); process.exit(1) })
```

실행:
```bash
pnpm run seed
```

## 6) 기본 CRUD 스모크 테스트

(1) Create — 이미 시드로 생성됨

(2) Read
```bash
node -e "const {PrismaClient}=require('@prisma/client');const p=new PrismaClient();(async()=>{console.log(await p.artist.findMany({include:{artworks:true}}));await p.$disconnect();})();"
```
(3) Update
```bash
node -e "const {PrismaClient}=require('@prisma/client');const p=new PrismaClient();(async()=>{await p.goods.update({where:{slug:'armo-mug'},data:{nameKo:'아르모 머그컵'}});console.log('updated');await p.$disconnect();})();"
```
(4) Delete
```bash
node -e "const {PrismaClient}=require('@prisma/client');const p=new PrismaClient();(async()=>{await p.faq.deleteMany();console.log('faq cleared');await p.$disconnect();})();"
```
* 위 Read/Update/Delete가 정상 동작하면 “로컬 DB에서 기본 CRUD 가능” 요건 충족!
* 길어지면 seed.cjs 같은 파일에 따로 넣고 node seed.cjs로 실행하는 게 훨씬 안정적이에요.

## 7) (선택) Prisma Studio로 시각 검증
```bash
pnpm prisma studio
```
* 브라우저에서 테이블/관계를 직접 확인하며 값 수정/삭제로 CRUD 재검증.

## 8) 체크리스트 — DoD 통과 여부

- pnpm prisma migrate dev 성공 (스키마 반영 OK)
- 로컬 DB에서 artists / artworks / goods / packages 등 기본 CRUD 동작
→ 스크립트/Studio로 읽기/수정/삭제 확인

