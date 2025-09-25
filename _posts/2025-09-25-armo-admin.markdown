---
title: “Armo 개발 단계 7 — Admin 최소 UI(Next.js /admin)”
description: “콘솔 대신 Next.js 내부에 /admin 인터페이스를 만들어 콘텐츠 생성·수정·발행을 할 수 있도록 합니다. 로그인, 리스트, 폼, 이미지 업로더, 체크리스트까지 구현하는 과정을 다룹니다.”
date: 2025-09-25 17:00:00 +0900
categories: [Armo, Frontend, Next.js, Admin, CMS]
tags: [Next.js, Admin, JWT, Fastify, Prisma, Supabase, CMS]
keywords: [“Next.js Admin”, “콘텐츠 관리”, “이미지 업로드”, “JWT 로그인”, “콘텐츠 발행”, “스토리 생성”, “SEO 체크리스트”]
author: "Jerome Na"
---

콘솔 대신 Next.js 내부에 /admin 인터페이스를 만들어 콘텐츠 생성·수정·발행을 할 수 있도록 합니다. 로그인, 리스트, 폼, 이미지 업로더, 체크리스트까지 구현하는 과정을 다룹니다.

---
## 목표
- /admin 경로에 최소한의 CMS 기능 구축
- 콘솔이나 DB 툴 없이도 콘텐츠 생성·수정·발행 가능
- 로그인(Owner만 접근 가능) → 리스트/검색/필터 → 생성·수정 폼 → 이미지 업로더 → 필수 필드 체크리스트 제공
- DoD: /admin에서 스토리 1건 생성 → 이미지 업로드 → 발행까지 성공

---

## 0. 왜 필요한가?

지금까지는 API와 DB를 통해서만 콘텐츠를 등록할 수 있었어요.

그러나 실제 운영에서는 매번 콘솔을 열어 SQL을 치거나 API를 호출하는 건 비효율적입니다.

👉 따라서 Next.js 앱 내부에 /admin 경로를 만들고, 브라우저 기반 최소 CMS를 제공합니다.

---

## 1. 로그인 (Owner 전용)

- JWT 기반 세션
- 발급: /api/auth/login (아이디/비번 → JWT)
- 저장: httpOnly Cookie
- 확인: Next.js middleware로 보호

```ts
// src/pages/api/auth/login.ts
import { sign } from 'jsonwebtoken'
import { serialize } from 'cookie'

export default async function handler(req, res) {
  const { email, password } = req.body
  if (email !== process.env.ADMIN_EMAIL || password !== process.env.ADMIN_PASS) {
    return res.status(401).json({ error: 'Unauthorized' })
  }
  const token = sign({ role: 'owner' }, process.env.JWT_SECRET!, { expiresIn: '1d' })
  res.setHeader('Set-Cookie', serialize('admin_token', token, { httpOnly: true, path: '/' }))
  res.json({ success: true })
}
```

```ts
// src/middleware.ts
import { NextResponse } from 'next/server'
import { verify } from 'jsonwebtoken'

export function middleware(req) {
  const token = req.cookies.get('admin_token')?.value
  if (req.nextUrl.pathname.startsWith('/admin')) {
    if (!token) return NextResponse.redirect(new URL('/login', req.url))
    try {
      verify(token, process.env.JWT_SECRET!)
      return NextResponse.next()
    } catch {
      return NextResponse.redirect(new URL('/login', req.url))
    }
  }
}
```

---

## 2. 리스트/검색/필터

- /admin/stories 페이지 → 스토리 목록
- 상태(draft/scheduled/published), 타입(brand/artwork/motif 등)으로 필터링 가능
- 검색창으로 제목 검색

```ts
// src/pages/admin/stories/index.tsx
import useSWR from 'swr'

export default function StoryList() {
  const { data } = useSWR('/api/stories')
  return (
    <div>
      <h1>스토리 관리</h1>
      <input placeholder="검색…" />
      <select><option>전체</option><option>draft</option>...</select>
      <table>
        <thead><tr><th>제목</th><th>상태</th><th>발행일</th></tr></thead>
        <tbody>
          {data?.map(story => (
            <tr key={story.id}>
              <td>{story.title}</td>
              <td>{story.status}</td>
              <td>{story.publishAt || '-'}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}
```

---

## 3. 생성·수정 폼

- /admin/stories/new, /admin/stories/[id]
- 제목, 본문, 타입, 이미지 업로드 입력 → POST/PUT API 호출

```ts
// src/pages/admin/stories/new.tsx
import { useState } from 'react'
import { useRouter } from 'next/router'

export default function NewStory() {
  const [title, setTitle] = useState('')
  const [body, setBody] = useState('')
  const router = useRouter()

  async function handleSave() {
    await fetch('/api/stories', {
      method: 'POST',
      headers: { 'Content-Type':'application/json' },
      body: JSON.stringify({ title, body }),
    })
    router.push('/admin/stories')
  }

  return (
    <div>
      <h1>새 스토리 작성</h1>
      <input value={title} onChange={e=>setTitle(e.target.value)} placeholder="제목"/>
      <textarea value={body} onChange={e=>setBody(e.target.value)} placeholder="본문"/>
      <button onClick={handleSave}>저장</button>
    </div>
  )
}
```

---

## 4. 이미지 업로더

- /api/media/upload 연결 (앞선 ⑥단계 API 재사용)
- 업로드 후 반환된 asset.id → 스토리 heroImage로 연결

```ts
function ImageUploader({ onUploaded }) {
  async function handleFile(e) {
    const file = e.target.files[0]
    const form = new FormData()
    form.append('file', file)
    form.append('bucket', 'common') // artworks/goods/common 중 선택

    const res = await fetch('/api/media/upload', { method: 'POST', body: form })
    const json = await res.json()
    onUploaded(json.asset.path)
  }
  return <input type="file" onChange={handleFile}/>
}
```

---

# 5. 체크리스트

스토리 저장/발행 전 필수 항목 검증:
- 제목 없음 ❌
- 썸네일 없음 ❌
- 슬러그 중복 ❌
- SEO 메타(description 150자 이내) 없음 ❌

UI 예시:

```html
<ul>
  <li className={title ? 'ok' : 'error'}>제목</li>
  <li className={hero ? 'ok' : 'error'}>썸네일</li>
  <li className={slugUnique ? 'ok' : 'error'}>슬러그 중복 체크</li>
  <li className={seo ? 'ok' : 'error'}>SEO 메타</li>
</ul>
```

---

## 6. 발행 버튼

- draft → scheduled/published 변경 API 호출
- 예약 발행은 publishAt 날짜 입력 → API /schedule 호출

```html
<button onClick={() => fetch(`/api/stories/${id}/publish`, { method:'POST' })}>
  지금 발행
</button>
```

---

## 7. DoD 검증

1.	/admin 접속 → 로그인 성공
2.	스토리 생성 페이지에서 제목+본문 작성 → 저장 → 목록 반영 확인
3.	이미지 업로더로 썸네일 추가
4.	체크리스트 OK → 발행 버튼 클릭
5.	상태가 published로 변경됨을 목록/DB에서 확인

✅ DoD: /admin에서 스토리 1건 생성 → 이미지 업로드 → 발행까지 완료
