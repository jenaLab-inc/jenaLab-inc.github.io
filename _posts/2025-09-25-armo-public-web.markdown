---
title: “Armo 개발 단계 8 — 퍼블릭 웹 페이지(SSR 최소, ISR/SSG 위주)”
description: “Armo 웹사이트 퍼블릭 영역을 Next.js ISR/SSG 기반으로 구축합니다. 홈, 스토리, 굿즈, 작가, 저널, 소개, 고객센터 페이지를 API 데이터와 이미지 LQIP로 최적화합니다.”
date: 2025-09-26 11:00:00 +0900
categories: [Armo, Frontend, Next.js, Web]
tags: [Next.js, ISR, SSG, 퍼블릭웹, LQIP, 블러프리뷰, CMS, Fastify]
keywords: [“Next.js ISR”, “SSG”, “Armo 웹사이트”, “퍼블릭 페이지”, “스토리 페이지”, “굿즈 페이지”, “작가 페이지”, “저널 페이지”, “이미지 최적화”, “LQIP”]
author: "Jerome Na"
---

Armo 웹사이트 퍼블릭 영역을 Next.js ISR/SSG 기반으로 구축합니다. 

홈, 스토리, 굿즈, 작가, 저널, 소개, 고객센터 페이지를 API 데이터와 이미지 LQIP로 최적화합니다.

---

## 목표
- IA대로 핵심 퍼블릭 페이지만 빠르게 노출
- SSR 최소화, ISR/SSG 위주로 SEO/성능 확보
- 홈(/), 스토리(/stories/[slug]), 굿즈(/goods/[slug]), 작가(/artists/[slug]), 저널(/journal/[slug]), 소개(/about), 고객센터(/help)
- API 호출 → ISR(예: 60초) 캐시
- 이미지 LQIP/블러 프리뷰 적용
- DoD: 배포본에서 상세 페이지들 200 응답 + 이미지 지연로딩 정상 작동

---

## 0. 왜 ISR/SSG인가?

Armo는 작품·스토리·굿즈를 콘텐츠 중심으로 보여주는 사이트입니다.

- SEO 노출이 중요 → 정적 HTML 필요
- 콘텐츠 변경은 잦지 않음 → 실시간 SSR은 과함
- Next.js ISR(Incremental Static Regeneration)으로 빌드 + 캐싱(60s 정도)이 적합

👉 결과: 방문자는 빠른 TTFB, 서버는 부하 최소화

---

## 1. 라우트 설계 (IA 기반)

IA(Information Architecture)에 맞춰 라우트를 구성합니다.

- / → 홈: 히어로 섹션 + 대표 스토리, 굿즈, 작가
- /stories/[slug] → 스토리 상세
- /goods/[slug] → 굿즈 상세
- /artists/[slug] → 작가 상세 (대표 작품 리스트 포함)
- /journal/[slug] → 저널 아티클 상세
- /about → 브랜드 소개
- /help → 고객센터/FAQ

추가적으로: /journal(목록), /stories(목록)도 확장 가능.

---

## 2. 데이터 페칭 (Next.js ISR)

API 서버(api/)로부터 데이터를 받아 ISR(예: 60초)로 캐싱합니다.

```tsx
// app/stories/[slug]/page.tsx
import { notFound } from 'next/navigation'

export const revalidate = 60 // ISR 60초

async function getStory(slug: string) {
  const res = await fetch(`${process.env.API_BASE}/api/stories/${slug}`, {
    next: { revalidate: 60 },
  })
  if (!res.ok) return null
  return res.json()
}

export default async function StoryPage({ params }: { params: { slug: string } }) {
  const story = await getStory(params.slug)
  if (!story) notFound()

  return (
    <article className="prose mx-auto">
      <h1>{story.title}</h1>
      <img src={story.heroImage} alt={story.title} className="rounded" loading="lazy" />
      <div dangerouslySetInnerHTML={{ __html: story.bodyHtml }} />
    </article>
  )
}
```

---

## 3. 홈(/) 페이지 구성

홈은 3가지 대표 콘텐츠를 뽑아 섹션으로 구성합니다.

```ts
// app/page.tsx
export const revalidate = 60

async function getHomeData() {
  const [stories, goods, artists] = await Promise.all([
    fetch(`${process.env.API_BASE}/api/stories?limit=3`).then(r=>r.json()),
    fetch(`${process.env.API_BASE}/api/goods?limit=3`).then(r=>r.json()),
    fetch(`${process.env.API_BASE}/api/artists?limit=3`).then(r=>r.json()),
  ])
  return { stories, goods, artists }
}

export default async function HomePage() {
  const { stories, goods, artists } = await getHomeData()
  return (
    <main>
      <section className="hero">Armo — Art + Motif + Emotion</section>
      <section>
        <h2>대표 스토리</h2>
        {stories.map((s:any)=><a key={s.slug} href={`/stories/${s.slug}`}>{s.title}</a>)}
      </section>
      <section>
        <h2>굿즈</h2>
        {goods.map((g:any)=><a key={g.slug} href={`/goods/${g.slug}`}>{g.nameKo}</a>)}
      </section>
      <section>
        <h2>작가</h2>
        {artists.map((a:any)=><a key={a.slug} href={`/artists/${a.slug}`}>{a.nameKo}</a>)}
      </section>
    </main>
  )
}
```

---

## 4. 이미지 LQIP/블러 프리뷰

- 이미지는 ⑥단계에서 **파생본(thumb/web/retina + webp/avif)** 이 이미 생성됩니다.
- 프론트에서는 blurDataURL을 적용해 LQIP(저화질 미리보기) UX를 제공합니다.

```ts
import Image from 'next/image'

export default function StoryImage({ src, alt, blur }: { src: string; alt: string; blur: string }) {
  return (
    <Image
      src={src}
      alt={alt}
      width={1280}
      height={720}
      placeholder="blur"
      blurDataURL={blur}
      loading="lazy"
      className="rounded"
    />
  )
}
```

blurDataURL은 워커에서 생성하거나 blurhash로 따로 만들어 DB에 저장할 수 있습니다.

---

# 5. 소개(/about), 고객센터(/help)

정적 콘텐츠이므로 SSG로만 처리합니다.

```ts
// app/about/page.tsx
export const revalidate = false // 변경 없음
export default function AboutPage() {
  return (
    <main className="prose mx-auto">
      <h1>Armo 소개</h1>
      <p>Armo는 작가의 작품을 굿즈와 스토리로 확장하는 럭셔리 아트 커머스 플랫폼입니다.</p>
    </main>
  )
}
```

---

## 6. DoD 체크리스트

	1.	/, /stories/[slug], /goods/[slug], /artists/[slug], /journal/[slug], /about, /help 모두 빌드 성공
	2.	Vercel/Fly.io 배포 후, 각 URL 200 응답 확인
	3.	이미지: next/image + blurDataURL → 로딩 시 블러 프리뷰 → 지연 로딩 정상
	4.	API 데이터 변경 시 → 60초 이내 ISR 리빌드로 반영 확인

✅ DoD 충족
