---
layout: post
ref: storida-toss-payments
title: "토스페이먼츠 빌링키 기반 구독 결제 시스템 설계"
date: 2026-02-09 14:00:00
categories: payments
author: jenalab
image: /assets/article_images/storida/payments.svg
image2: /assets/article_images/storida/payments.svg
---

## SaaS 구독 결제의 요건

수익 모델은 월 구독이다. 필요한 요건은 네 가지다.

1. 사용자가 카드를 한 번만 등록하면 매월 자동 결제
2. 요금제 변경 시 즉시 차액 결제
3. 해지 시 현재 기간 만료까지 서비스 유지
4. 결제 실패 시 재시도 후 자동 만료

토스페이먼츠 빌링키 방식으로 구현했다.

## 빌링키 결제 플로우

카드 정보를 직접 저장하지 않는다. 토스페이먼츠가 발급한 빌링키로 결제한다.

```
1. Web ──────────────▶ 토스 결제창 (카드 등록)
                            │
2. 토스 ─── authKey ──────▶ Web (successUrl 리디렉트)
                            │
3. Web ──── confirm ──────▶ Backend
                            │
4. Backend                  │
   ├── authKey → 빌링키 발급 (토스 API)
   ├── 빌링키로 첫 결제 실행 (토스 API)
   ├── subscriptions 레코드 생성
   ├── payments 레코드 생성
   └── 토큰 지급 + credit_logs 기록
                            │
5. Scheduler (매월) ──────▶ 빌링키로 자동 갱신 결제
```

### 핵심: Backend가 모든 결제 로직을 처리

Web은 토스 결제창을 열고 `authKey`를 받는 역할만 한다. 빌링키 발급, 실제 결제, DB 저장은 모두 Backend에서 수행한다. 이는 **결제 무결성**을 보장하기 위한 설계다.

## DB 설계

### subscriptions 테이블

구독 정보는 사용자당 하나만 존재한다(UNIQUE 제약). 빌링키, 고객키, 상태(active/cancelled/past_due/expired), 현재 결제 기간을 저장한다. 사용자당 하나의 활성 구독만 허용하여 중복 구독을 원천 차단한다.

### payments 테이블

결제 기록은 구독과 연결되며, 토스 결제 고유키, 주문번호, 금액, 상태(pending/success/failed/refunded), 영수증 URL을 저장한다.

## API 설계

| 메서드 | 경로 | 설명 |
|---|---|---|
| POST | `/billing/checkout` | 결제 준비 (orderId, amount 반환) |
| POST | `/billing/confirm` | 구독 확정 (빌링키 발급 + 첫 결제) |
| GET | `/billing/me` | 내 구독 조회 |
| POST | `/billing/cancel` | 구독 해지 |
| POST | `/billing/change` | 요금제 변경 |

### confirm API — 가장 복잡한 엔드포인트

`confirm`은 하나의 요청에서 4가지 외부 호출과 DB 트랜잭션을 수행한다.

```
// confirmSubscription 처리 흐름
// 1. authKey로 토스 빌링키 발급
// 2. 빌링키로 첫 결제 실행
// 3. DB 트랜잭션으로 아래를 원자적으로 수행:
//    - 구독 레코드 생성 (active 상태, 1개월 기간)
//    - 결제 기록 저장
//    - 사용자 토큰 지급
//    - 토큰 이력 기록
```

트랜잭션으로 묶음으로써, 결제는 성공했는데 토큰이 지급되지 않는 상황을 방지한다.

## Scheduler 자동 갱신

Scheduler는 1시간 간격으로 자동 갱신을 처리한다.

```
chargeExpiredSubscriptions (1시간 간격)
    │
    ├── status = 'active'이고
    │   current_period_end < now()인 구독 조회
    │
    ├── 빌링키로 자동 결제 시도
    │   ├── 성공 → 다음 기간 갱신 + 토큰 초기화
    │   └── 실패 → status = 'past_due'
    │
expirePastDueSubscriptions (1시간 간격)
    │
    └── past_due 상태가 3일 초과 → status = 'expired'

expireCancelledSubscriptions (1시간 간격)
    │
    └── cancelled이고 기간 만료 → status = 'expired'
```

### 결제 실패 처리 전략

```
active ──결제 실패──▶ past_due ──3일 경과──▶ expired
                         │
                    결제 재시도 성공
                         │
                         ▼
                      active
```

`past_due` 상태는 사용자에게 결제 수단 확인을 요청하는 유예 기간이다. 3일 동안 매시간 재시도하며, 복구되지 않으면 자동 만료한다.

## 요금제 변경 설계

요금제 변경(업그레이드/다운그레이드)은 **즉시 결제 + 즉시 적용** 방식이다.

```
// 요금제 변경 처리 흐름
// 1. 현재 구독과 새 요금제 조회
// 2. 빌링키로 새 요금제 금액 즉시 결제
// 3. 구독 정보 업데이트 (새 요금제, 새 기간)
// 4. 토큰을 새 요금제 할당량으로 즉시 갱신
```

일할 계산(proration)은 제외했다. 사용자 규모가 작은 지금, 즉시 결제가 더 단순하고 이해하기 쉽다.

## 주의사항

1. **토스 SDK v2**: `tossPayments.payment({ customerKey }).requestBillingAuth()` — `billing` 속성이 없다
2. **Next.js Suspense**: `useSearchParams()` 사용 시 `<Suspense>` boundary 필수
3. **환경변수 분리**: Client Key(Web)와 Secret Key(Backend/Scheduler)를 분리한다

다음 글은 Vercel 60초 타임아웃을 우회하는 Job Queue 비동기 생성 시스템을 다룬다.
