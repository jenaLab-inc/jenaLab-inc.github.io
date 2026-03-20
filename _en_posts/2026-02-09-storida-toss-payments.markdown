---
layout: post
ref: storida-toss-payments
title: "Toss Payments Billing Key — Designing a Subscription Payment System"
date: 2026-02-09 14:00:00
categories: payments
author: jenalab
image: /assets/article_images/storida/payments.svg
image2: /assets/article_images/storida/payments.svg
---

## Subscription Requirements

The revenue model is monthly subscriptions. Four requirements.

1. Register a card once. Auto-charge every month.
2. Plan changes trigger immediate payment.
3. Cancellation keeps service active until the current period ends.
4. Failed payments retry. Unrecovered failures expire the subscription.

Toss Payments billing key system handles all four.

## Billing Key Flow

Card details never touch our servers. Toss issues a billing key that represents the card.

```
1. Web ──────────────▶ Toss payment window (card registration)
                            │
2. Toss ─── authKey ──────▶ Web (redirects to successUrl)
                            │
3. Web ──── confirm ──────▶ Backend
                            │
4. Backend
   ├── authKey → billing key issuance (Toss API)
   ├── billing key → first charge (Toss API)
   ├── Create subscriptions record
   ├── Create payments record
   └── Grant tokens + credit_logs record
                            │
5. Scheduler (monthly) ───▶ Auto-renewal via billing key
```

### Key Principle: Backend Owns All Payment Logic

Web opens the Toss payment window and receives the `authKey`. That is all Web does. Billing key issuance, charging, DB writes — Backend handles everything. This guarantees payment integrity.

## Database Design

### Subscriptions Table

Each subscription record tracks the owner, the selected plan, the Toss billing and customer keys, a status (active, cancelled, past_due, or expired), the current billing period boundaries, and a cancellation timestamp. A unique constraint on the user ensures one active subscription per user. Duplicate subscriptions are impossible at the database level.

### Payments Table

Each payment record links to a user, a subscription, and the corresponding Toss transaction identifiers (payment key and order ID). It stores the charge amount, a status (pending, success, failed, or refunded), and a receipt URL. Every charge creates a payment record. The Toss payment key links back to the Toss transaction for reconciliation.

## API Design

| Method | Path | Description |
|---|---|---|
| POST | `/billing/checkout` | Payment preparation (returns orderId, amount) |
| POST | `/billing/confirm` | Subscription confirmation (billing key + first charge) |
| GET | `/billing/me` | Current subscription info |
| POST | `/billing/cancel` | Cancel subscription |
| POST | `/billing/change` | Change plan |

### The Confirm Endpoint — Most Complex

`confirm` executes 4 external calls and a DB transaction in a single request.

```
// 1. Exchange authKey for a billing key via Toss API
// 2. Execute the first charge via Toss API
// 3. All within a single DB transaction:
//    a. Create subscription record (active, 1-month period)
//    b. Record the payment
//    c. Grant plan tokens to the user
//    d. Log the credit change
```

The transaction ensures atomicity. A successful charge always results in a subscription record and token grant. No partial state.

## Scheduler Auto-Renewal

The Scheduler checks for expired subscriptions every hour.

```
chargeExpiredSubscriptions (hourly)
    │
    ├── Find subscriptions where
    │   status = 'active' AND
    │   current_period_end < now()
    │
    ├── Charge via billing key
    │   ├── Success → renew period + reset tokens
    │   └── Failure → status = 'past_due'

expirePastDueSubscriptions (hourly)
    │
    └── past_due for 3+ days → status = 'expired'

expireCancelledSubscriptions (hourly)
    │
    └── cancelled AND period ended → status = 'expired'
```

## Subscription Status Flow

Four states. Two paths to expiration.

```
                    charge failed
active ─────────────────────────▶ past_due ──3 days──▶ expired
   │                                  │
   │                            retry succeeds
   │                                  │
   │                                  ▼
   │                               active
   │
   │  user cancels
   └──────────────▶ cancelled ──period ends──▶ expired
                    (service continues)
```

`past_due` is a grace period. The system retries every hour for 3 days. The user sees a prompt to update their payment method. After 3 days without recovery, the subscription expires.

`cancelled` preserves service until the paid period ends. The user already paid for this month. They get the full month.

## Plan Change Design

Plan changes use immediate payment and immediate application. No proration.

```
// 1. Look up the user's current subscription and the new plan
// 2. Charge the new plan price immediately via Toss billing key
// 3. Update the subscription to the new plan with a fresh 1-month period
// 4. Reset the user's token balance to the new plan's monthly allocation
```

Proration adds complexity. At the current user scale, immediate charge is simpler and easier to explain. Users understand "you pay the new price now and get a fresh month."

## Technical Notes

### Toss SDK v2 Billing API

The v2 SDK has no `billing` property on the `tossPayments` object. Instead, billing auth is requested through the `payment()` method with a customer key. The method accepts card type, success URL, and fail URL as parameters.

### Next.js useSearchParams and Suspense

The success redirect URL contains query parameters (`authKey`, `customerKey`). `useSearchParams()` in Next.js requires the component to be wrapped in a React `<Suspense>` boundary. Without it, the build fails.

### Environment Variable Separation

| Variable | Location | Purpose |
|---|---|---|
| Client Key | Web (public) | Opens Toss payment window |
| Secret Key | Backend + Scheduler (server only) | Billing key issuance, charges |

The Client Key is safe to expose. The Secret Key never leaves the server.

Next: the DB-based job queue and async generation system.
