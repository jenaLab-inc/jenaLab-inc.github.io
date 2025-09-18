---
title: "Armo 개발 단계 #3: Fastify 기반 API 서버 스캐폴딩"
date: 2025-09-18
description: "Armo 프로젝트에서 API 서버를 Fastify로 구축하는 과정. 보안·에러 핸들링·로그 설정부터 JWT 인증까지, 실무적인 스캐폴딩 과정을 단계별로 설명합니다."
keywords: ["Armo", "Fastify", "API 서버", "Node.js", "Fly.io", "스캐폴딩", "JWT 인증", "백엔드 개발", "Supabase", "pg-boss"]
author: "Jerome Na"
tags: ["개발 단계", "Fastify", "API 서버", "스캐폴딩"]
---

Armo 프로젝트의 백엔드 중심은 Fastify 기반 API 서버입니다.  
이전 단계에서 `/healthcheck` 엔드포인트까지 만들어 서버가 정상 기동되는 걸 확인했으니, 이번에는 **보안·에러 처리·JWT 인증**을 추가해 실무에 쓸 수 있는 기본 골격을 갖추겠습니다.

---

## 목표

- Fastify 서버에 보안 미들웨어(CORS, Helmet) 적용  
- Zod를 이용한 DTO 검증 및 에러 포맷 통일  
- JWT 기반 인증을 추가해 보호된 라우트 구현  

---

## 작업 순서

### 1. 필수 패키지 설치
```bash
pnpm add fastify-cors fastify-helmet @fastify/jwt zod
```

### 2. 보안 미들웨어 적용
```ts
// index.ts
import cors from "@fastify/cors";
import helmet from "@fastify/helmet";

await app.register(cors, { origin: "*" });
await app.register(helmet);
```

### 3. 에러 포맷 & DTO 검증
```ts
import { z } from "zod";

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(6),
});

app.setErrorHandler((error, req, reply) => {
  req.log.error(error);
  reply.status(400).send({
    success: false,
    message: error.message,
  });
});
```

### 4. JWT 인증 추가
```ts
import fastifyJwt from "@fastify/jwt";

app.register(fastifyJwt, { secret: process.env.JWT_SECRET! });

app.decorate("authenticate", async (req, reply) => {
  try {
    await req.jwtVerify();
  } catch (err) {
    reply.send(err);
  }
});

app.post("/login", async (req, reply) => {
  const { email, password } = req.body as any;
  if (email === "owner@armo.dev" && password === "admin") {
    const token = app.jwt.sign({ role: "owner" });
    return { token };
  }
  return reply.status(401).send({ error: "Unauthorized" });
});

app.get("/protected", { preValidation: [app.authenticate] }, async () => {
  return { msg: "비공개 데이터 접근 성공" };
});
```

---

## DoD (Definition of Done)
- /healthcheck 정상 동작 확인
- 로그인 → JWT 발급 → 보호된 라우트 접근 성공 확인
- 잘못된 입력 시 일관된 JSON 에러 응답 확인
