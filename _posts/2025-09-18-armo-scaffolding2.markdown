---
title: "Armo 개발 단계 #3: Fastify API 서버 스캐폴딩 실습 튜토리얼"
date: 2025-09-18
description: "Fastify API 서버 스캐폴딩을 실습형 튜토리얼로 따라하며 DoD(Definition of Done) 기준까지 완성해보는 과정. 보안(CORS, Helmet), 에러 핸들링, Zod DTO 검증, JWT 로그인, argon2 해시 적용까지 단계별로 진행합니다."
keywords: ["Armo","Fastify","API 서버","스캐폴딩","JWT","argon2","Zod","보안","DoD","Node.js"]
author: "Jerome Na"
tags: ["개발 단계","Fastify","백엔드","실습 튜토리얼"]
---

Armo 프로젝트에서 백엔드의 첫걸음은 **Fastify 기반 API 서버 스캐폴딩**입니다.  
이번 글에서는 **실습 튜토리얼** 형식으로, 순서대로 따라 하면 DoD(Definition of Done) 기준까지 완성할 수 있게 정리했습니다.

> 예시 소스는 전체 소스가 아닙니다. 구조만 보여주기 위한 소스이기 때문에 ...으로 되어 있는 부분은 각자에 맞게 수정하셔야 합니다.

---

## 목표 (DoD)
- `/healthcheck` 엔드포인트 200 OK  
- CORS/Helmet 보안 미들웨어 적용  
- 에러 발생 시 JSON 포맷 통일  
- JWT 로그인 성공 후 토큰 발급  
- 보호 라우트 접근 성공  
- `admin` 비밀번호는 **argon2 해시**로 안전하게 관리  

---

## 전체 구성
```bash
src/
 ├─ config/
 │   ├─ env.ts          // Zod로 환경변수 로드/검증/타입
 │   └─ index.ts        // config 객체 export (싱글턴)
 ├─ modules/
 │   ├─ auth/
 │   │   ├─ auth.routes.ts
 │   │   └─ auth.schemas.ts   // Zod 스키마
 │   └─ content/
 │       ├─ content.routes.ts
 │       └─ content.schemas.ts
 ├─ plugins/
 │   ├─ errorHandler.ts  // 전역 에러 핸들러
 │   ├─ jwt.ts          // JWT 등록 + authenticate 훅
 │   ├─ cors.ts         // ALLOWED_ORIGINS 적용
 │   ├─ rateLimit.ts    // RATE_LIMIT_MAX/TIME_WINDOW 적용
 │   └─ logger.ts       // LOG_LEVEL 반영(선택)
 └─ index.ts            // Fastify 부트스트랩에서 config 사용
```
---
## 1. 프로젝트 부트스트랩
```bash
cd api
pnpm init
pnpm add fastify @fastify/cors @fastify/helmet @fastify/jwt @fastify/cookie zod argon2
pnpm add -D typescript ts-node-dev @types/node
```

tsconfig.json 기본 설정:
```ts
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```
package.json 스크립트:
```json
{
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/index.js"
  }
}
```
---
## 2. 환경변수 스키마 (config/env.ts)
```ts
import { z } from "zod";
import * as dotenv from "dotenv";
dotenv.config();

const envSchema = z.object({
  NODE_ENV: z.enum(["development","production","test"]).default("development"),
  PORT: z.coerce.number().default(3000),
  HOST: z.string().default("0.0.0.0"),
  JWT_SECRET: z.string().default("dev-secret-key"),
  JWT_EXPIRES_IN: z.string().default("7d"),
  ALLOWED_ORIGINS: z.string().default("http://localhost:3000").transform(val => val.split(","))
});
export const env = envSchema.parse(process.env);
```
---
## 3. 서버 부트스트랩 (index.ts)
```ts
import Fastify from "fastify";
import helmet from "@fastify/helmet";
import cors from "@fastify/cors";
import { env } from "./config/env";
import jwtPlugin from "./plugins/jwt";
import { errorHandler } from "./plugins/errorHandler";
import authRoutes from "./modules/auth/auth.routes";

async function bootstrap() {
  const app = Fastify({ logger: true });

  await app.register(helmet);
  await app.register(cors, { origin: env.ALLOWED_ORIGINS, credentials: true });
  await app.register(jwtPlugin);
  await app.register(authRoutes);

  app.get("/healthcheck", async () => ({ status: "ok" }));
  app.setErrorHandler(errorHandler);

  await app.listen({ host: env.HOST, port: env.PORT });
}
bootstrap();
```
---
## 4. 에러 핸들러 (plugins/errorHandler.ts)
```ts
import { FastifyError, FastifyReply, FastifyRequest } from "fastify";
import { ZodError } from "zod";

export const errorHandler = (
  error: FastifyError | ZodError | Error,
  request: FastifyRequest,
  reply: FastifyReply
) => {
  request.log.error(error);

  if (error instanceof ZodError) {
    return reply.status(422).send({
      success: false,
      error: "VALIDATION_ERROR",
      details: error.issues,
    });
  }

  const statusCode = (error as FastifyError).statusCode || 500;
  const message = error.message || "Internal Server Error";

  return reply.status(statusCode).send({
    success: false,
    error: "INTERNAL_ERROR",
    message: process.env.NODE_ENV === "production" ? "Something went wrong" : message,
  });
};
```
---
## 5. JWT 플러그인 (plugins/jwt.ts)
```ts
// src/plugins/jwt.ts
import type { FastifyInstance, FastifyReply, FastifyRequest } from "fastify";
import fp from "fastify-plugin";
import jwt from "@fastify/jwt";
import cookie from "@fastify/cookie";
import { getConfig } from "../config";

declare module "fastify" {
  interface FastifyInstance {
    authenticate: (req: FastifyRequest, rep: FastifyReply) => Promise<void>;
    authorizeRole: (role: string) => (req: FastifyRequest, rep: FastifyReply) => Promise<void>;
  }
  interface FastifyRequest {
    user: { sub: string; role: string } // 토큰 페이로드 타입
  }
}

export default fp(async function jwtPlugin(app: FastifyInstance) {
  const { JWT_SECRET, JWT_EXPIRES_IN, NODE_ENV } = getConfig();

  // 쿠키(선택) + JWT 등록
  await app.register(cookie, {
    secret: undefined, // 서명 쿠키 쓰려면 secret 지정
    hook: "onRequest",
  });

  await app.register(jwt, {
    secret: JWT_SECRET,
    cookie: {
      cookieName: "access_token", // 쿠키 토큰 이름
      signed: false,
    },
    sign: {
      expiresIn: JWT_EXPIRES_IN, // "7d" 등
    },
  });

  // 공통: 토큰 파싱(쿠키 우선, 없으면 Authorization)
  function extractToken(req: FastifyRequest): string | null {
    const fromCookie = req.cookies?.access_token;
    if (fromCookie) return fromCookie;

    const auth = req.headers.authorization;
    if (auth && auth.startsWith("Bearer ")) {
      return auth.substring("Bearer ".length);
    }
    return null;
  }

  // 인증 미들웨어
  app.decorate("authenticate", async (req: FastifyRequest, rep: FastifyReply) => {
    const token = extractToken(req);
    if (!token) {
      return rep.status(401).send({ success: false, error: "UNAUTHORIZED", message: "Missing token" });
    }
    try {
      req.user = await app.jwt.verify(token);
    } catch (e) {
      return rep.status(401).send({ success: false, error: "UNAUTHORIZED", message: "Invalid or expired token" });
    }
  });

  // 권한 미들웨어
  app.decorate("authorizeRole", (role: string) => {
    return async (req: FastifyRequest, rep: FastifyReply) => {
      if (!req.user) {
        return rep.status(401).send({ success: false, error: "UNAUTHORIZED", message: "Not authenticated" });
      }
      if (req.user.role !== role) {
        return rep.status(403).send({ success: false, error: "FORBIDDEN", message: "Insufficient role" });
      }
    };
  });

  app.log.info(`JWT plugin ready (env=${NODE_ENV}, exp=${JWT_EXPIRES_IN})`);
});
```
---
## 6. Auth 모듈

(1) 스키마 (modules/auth/auth.schemas.ts)
```ts
import { z } from "zod";
export const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(4),
});
```
(2) 서비스 (modules/auth/auth.service.ts)
```ts
// src/modules/auth/auth.service.ts
import argon2 from "argon2";

type User = {
  id: string;
  email: string;
  role: "owner" | "editor";
  // 저장시에는 해시만! (plaintext 금지)
  passwordHash: string;
};

// MVP: 하드코딩 계정 (owner/admin)
const demoUser: User = {
  id: "u_owner_1",
  email: "owner@armo.dev",
  role: "owner",
  // "admin"를 argon2로 해시한 예제 (런타임에 생성해도 됨)
  passwordHash: "$argon2id$v=19$m=65536,t=3,p=1$pXgQ9Fz9oB6rX8h0W3Hk0g$1m2zH+3rVw6F9Y6tKvzvS9L+1wQH6Eo2mS0m6Zq0zA8"
};

export async function findUserByEmail(email: string): Promise<User | null> {
  if (email === demoUser.email) return demoUser;
  return null;
}

export async function verifyPassword(plain: string, hash: string): Promise<boolean> {
  try {
    return await argon2.verify(hash, plain);
  } catch {
    return false;
  }
}
```
(3) 라우트 (modules/auth/auth.routes.ts)
```ts
// src/modules/auth/auth.routes.ts
import type { FastifyInstance } from "fastify";
import { loginSchema } from "./auth.schemas";
import { findUserByEmail, verifyPassword } from "./auth.service";
import { getConfig } from "../../config";

export default async function authRoutes(app: FastifyInstance) {
  const { JWT_EXPIRES_IN, NODE_ENV } = getConfig();

  app.post("/auth/login", async (req, rep) => {
    const parsed = loginSchema.safeParse(req.body);
    if (!parsed.success) {
      // 전역 errorHandler가 있다면 throw parsed.error
      return rep.status(422).send({ success: false, error: "VALIDATION_ERROR", message: "Invalid payload", details: parsed.error.issues });
    }

    const { email, password } = parsed.data;

    const user = await findUserByEmail(email);
    if (!user) {
      return rep.status(401).send({ success: false, error: "UNAUTHORIZED", message: "Invalid credentials" });
    }

    const ok = await verifyPassword(password, user.passwordHash);
    if (!ok) {
      return rep.status(401).send({ success: false, error: "UNAUTHORIZED", message: "Invalid credentials" });
    }

    // 토큰 발급 (sub: 사용자 식별자)
    const token = await rep.jwtSign({ sub: user.id, email: user.email, role: user.role }, { expiresIn: JWT_EXPIRES_IN });

    // 1) Bearer 반환(모바일/서드파티 클라이언트에 적합)
    // 2) 또는 HttpOnly 쿠키 설정(동일 도메인 Admin에 적합) — 둘 다 제공 가능
    rep.setCookie("access_token", token, {
      httpOnly: true,
      secure: NODE_ENV === "production",
      sameSite: "lax",
      path: "/",
      maxAge: 60 * 60 * 24 * 7, // 7d (JWT_EXPIRES_IN과 맞추기)
    });

    return rep.send({
      success: true,
      token, // 쿠키만 쓰려면 이 줄은 제거 가능
      user: { id: user.id, email: user.email, role: user.role },
    });
  });

  // 내 정보
  app.get("/auth/me", { preValidation: [app.authenticate] }, async (req) => {
    const { sub, email, role } = req.user;
    return { success: true, user: { id: sub, email, role } };
  });

  // 로그아웃(쿠키를 쓸 경우)
  app.post("/auth/logout", async (req, rep) => {
    rep.clearCookie("access_token", { path: "/" });
    return { success: true };
  });

  // 샘플 보호 라우트
  app.get("/protected", { preValidation: [app.authenticate] }, async () => {
    return { success: true, data: "private" };
  });

  // 역할 보호 예시: owner만 접근
  app.get("/admin/only", { preValidation: [app.authenticate, app.authorizeRole("owner")] }, async () => {
    return { success: true, data: "owner secret" };
  });
}
```
---
## 7. 실습 검증

① 서버 실행
```bash
pnpm dev
```

② Healthcheck
```bash
curl http://localhost:3000/healthcheck
# {"status":"ok"}
```

③ 로그인
```bash
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@armo.dev","password":"admin"}'
```

응답:
```json
{ "success": true, "token": "eyJhbGciOi..." }
```

④ 보호 라우트
```bash
curl http://localhost:3000/protected \
  -H "Authorization: Bearer <위에서 받은 토큰>"
```

응답:
```json
{ "success": true, "user": { "sub": "u_owner_1", "role": "owner" } }
```
---
## 최종 DoD 체크
- /healthcheck 200 OK
- CORS/Helmet 적용 확인
- 잘못된 요청 시 JSON 에러 응답
- 로그인 → JWT 발급 성공
- 토큰으로 보호 라우트 접근 성공
- admin 비밀번호는 argon2 해시로 저장