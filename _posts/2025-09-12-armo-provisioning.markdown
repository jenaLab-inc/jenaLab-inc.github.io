---
title: "Armo 개발 단계 - 2. 인프라 1차 프로비저닝"
description: "Armo 웹사이트 구축 과정 중 2단계, 인프라 1차 프로비저닝 단계에 대해 Supabase, Vercel, Fly.io 설정 순서와 체크리스트를 소개합니다."
keywords: ["Armo", "웹사이트 개발", "인프라", "프로비저닝", "Supabase", "Vercel", "Fly.io", "Next.js", "Fastify"]
date: 2025-09-12
author: "Jerome Na"
---

Armo 프로젝트의 두 번째 단계는 인프라 1차 프로비저닝입니다.

개발자가 본격적으로 코드를 작성하기 전에 DB, 스토리지, 호스팅 환경을 준비하여 언제든 애플리케이션을 붙일 수 있는 기반을 만드는 과정입니다.

이 글에서는 Supabase, Vercel, Fly.io를 활용하여 Postgres + Storage + 웹 + API 인프라를 최소한으로 세팅하는 순서를 안내합니다.

⸻

목표
- 즉시 연결 가능한 데이터베이스와 스토리지 확보
- 웹/백엔드 호스팅을 위한 배포 환경 오픈
- 도메인 연결 전까지는 임시 URL로 테스트 가능

완료 기준(DoD):
- Vercel Preview URL 정상 동작
- Fly.io API 서버 /health 요청에 200 응답

⸻

## 1. Supabase 프로젝트 생성

1. Supabase 콘솔 접속 → 새 프로젝트 생성
2. 리소스 설정:
    - Database: Postgres (프로젝트별 자동 생성)
	- Storage: 3개 버킷 준비
	    - artworks (작품 이미지)
	    - goods (굿즈 이미지)
	    - common (공통 리소스)
	- 권한 정책:
	    - Public Read
	    - Authenticated Write
3. 환경변수 확보:
    - DATABASE_URL, SUPABASE_KEY, SUPABASE_URL

⸻

## 2. Vercel 프로젝트 생성 (Frontend)

1. Vercel 대시보드에서 New Project → GitHub Repo 연결
    - web 디렉토리(Next.js) 선택
2. 빌드 세팅:
    - Framework: Next.js
	- Root: ./web
3. 환경변수 입력:
	- NEXT_PUBLIC_SUPABASE_URL
	- NEXT_PUBLIC_SUPABASE_ANON_KEY
4. Preview URL 확인

* https://armo-web.vercel.app 와 같은 주소 자동 발급

## 3. Fly.io 앱 생성 (Backend API)

1.	Fly.io CLI 설치 → 로그인
```bash
flyctl auth login
```
2.	새 앱 생성:
```bash
cd api
flyctl launch
```
- 이름: armo-api
- 리전: 가까운 리전 선택 (예: nrt 도쿄)

3.	헬스체크 엔드포인트 준비:
```js
fastify.get("/health", async () => ({ status: "ok" }));
```
4.	배포 후 테스트:
```bash
curl https://armo-api.fly.dev/health
```

* 200 응답 나오면 성공

## 체크리스트
- Supabase Postgres/Storage 연결 완료
- Vercel 프론트엔드 Preview URL 정상 동작
- Fly.io API 서버 /health OK
- 환경변수 정리 및 GitHub Secrets에 등록

## 마무리

이 단계까지 완료하면 Armo 프로젝트의 기본 인프라 뼈대가 준비됩니다.