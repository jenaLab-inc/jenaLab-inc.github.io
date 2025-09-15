---
title: "Armo 프로젝트 인프라 실습기 — Vercel, Supabase, Fly.io 환경 설정과 오류 해결"
description: "Armo 웹사이트 구축 과정에서 진행한 Vercel, Supabase, Fly.io 인프라 세팅과 환경변수, SSL/TLS, 배포 오류 해결 방법을 단계별로 기록한 실습기."
author: "Jerome Na"
date: 2025-09-14
tags: ["Armo", "인프라", "Vercel", "Supabase", "Fly.io", "Next.js", "Fastify", "SSL 오류", "배포 오류"]
---

 
_Vercel · Supabase · Fly.io 설정과 오류 해결_

Armo 프로젝트는 **작가 작품을 굿즈화하고, 럭셔리한 브랜드 경험으로 연결하는 웹사이트**입니다.

이번 글에서는 실제 개발 단계에서 수행한 **인프라 세팅 과정**과, 그 과정에서 만난 **오류 및 해결 방법**을 정리합니다.

---

## 1. 인프라 1차 프로비저닝

Armo의 아키텍처는 **프론트엔드(Next.js) + 백엔드(Fastify API) + Supabase(Postgres, Storage)**로 시작합니다.

- **Supabase**
  - Database(Postgres) 생성
  - Storage 버킷 생성: `artworks`, `goods`, `common`
  - 권한 정책:  
    - `Public Read` (모두 읽기 가능)  
    - `Authenticated Write` (로그인 사용자만 쓰기 가능)

- **Vercel**
  - `armo-web` 프로젝트 생성
  - Preview URL로 Next.js 배포 확인

- **Fly.io**
  - `armo-api` 프로젝트 생성
  - Fastify API 컨테이너 배포
  - `/health` 엔드포인트에서 200 응답 확인

---

## 2. 환경변수 설정

실행에 필요한 환경변수:

```env
DATABASE_URL=postgres://...
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_KEY=...
JWT_SECRET=랜덤_생성값
```
- DATABASE_URL → Supabase 프로젝트에서 제공
- SUPABASE_URL, SUPABASE_KEY → Supabase → Project Settings > API에서 확인
- JWT_SECRET → 직접 생성해야 하는 값

```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

---

## 3. 배포 오류와 해결 방법

⚠ Vercel 404: NOT_FOUND
- 원인: pages/index.tsx와 app/page.tsx가 충돌 → 둘 중 하나만 유지해야 함.
- 해결: App Router를 쓰는 경우 pages/ 디렉토리 삭제.

⚠ GitHub Actions — SHA Pinning 오류
```bash
Error: The actions actions/checkout@v4 ... must be pinned to a full-length commit SHA.
```

- 원인: GitHub 리포지토리 보안 정책으로, 액션 버전을 SHA로 고정해야 함.
- 해결: 다음과 같이 수정

```yaml
- uses: actions/checkout@v4
  # ↓ 변경
- uses: actions/checkout@3df4a4ef3b3cfd8457c5d2a...
```

⚠ Fly.io API 기동 오류

```bash
[PM07] failed to change machine state: machine still active, refusing to start
```

- 원인: 기존 프로세스가 종료되지 않은 상태에서 새로 시작을 시도함.
- 해결:

```bash
fly machines stop <machine_id>
fly machines start <machine_id>
```

⚠ Fastify 라우트 중복 오류
```bash
code: 'FST_ERR_DUPLICATED_ROUTE'
```

- 원인: 동일한 경로(/health 등)를 여러 번 등록
- 해결: 라우트 정의를 점검하고 중복된 핸들러 제거.

⚠ cURL / OpenSSL SSL 오류
```bash
curl: (35) LibreSSL/3.3.6: error:1404B42E:SSL routines:...
```
- 원인: macOS 기본 curl이 구버전 LibreSSL을 사용해 TLS1.3 호환 문제 발생.
- 해결: Homebrew로 최신 curl 설치
```bash
brew install curl
echo 'export PATH="/opt/homebrew/opt/curl/bin:$PATH"' >> ~/.zshrc
exec $SHELL -l
curl -v https://armo-api.fly.dev/health
```

⚠ 그래도 SSL 오류가 난다면..
```bash
flyctl ssh console -a armo-api
```
- -a armo-api : 앱 이름
- 실행하면 컨테이너 내부 쉘로 들어가요.

```bash
apk add --no-cache curl     # Alpine 기반
apt-get update && apt-get install -y curl   # Debian/Ubuntu 기반
```

```bash
# 내부 포트 확인 (보통 Fly는 PORT=8080을 할당)
echo $PORT
# 또는 기본값 8080 사용

curl -v http://localhost:8080/health
```
- 200 OK가 나오면 → 앱은 정상이고, 외부에서 HTTPS가 안 되는 건 TLS/네트워크/DNS 문제임.

---
## 4. 헬스체크 & 배포 후 테스트

- API 헬스체크
```bash
curl -v https://armo-api.fly.dev/health
```
→ 200 OK 응답 확인

- 웹 배포 확인
- Vercel Preview URL 접속
- /stories/[slug], /goods/[id] 페이지 정상 동작 여부 확인
- 실서비스 점검
- Admin 로그인 → 콘텐츠 생성/수정 가능 여부
- 이미지 업로드 → Supabase Storage에 저장 확인
- /health 엔드포인트 모니터링 → Fly.io 자동 재시작 확인

## 5. 정리

이번 실습에서 겪은 핵심 포인트:
- 환경변수: Supabase 제공 값 + 직접 생성하는 JWT_SECRET 필요
- 배포 충돌: Next.js App/Pages 디렉토리 중복, GitHub Actions SHA pinning 문제
- Fly.io 문제: 머신 상태 충돌 시 stop → start로 해결
- SSL 문제: macOS 기본 curl 대신 Homebrew curl 사용
- Fastify 문제: 라우트 중복 제거

Armo 프로젝트 인프라는 Vercel + Supabase + Fly.io 조합으로 MVP 단계를 무리 없이 진행 가능했습니다.
