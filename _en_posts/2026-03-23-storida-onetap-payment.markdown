---
layout: post
ref: storida-onetap-payment
title: "One-Tap Generation and Per-Story Payment — From Subscription to Single Payment"
date: 2026-03-23 14:00:00
categories: architecture
author: jenalab
lang: en
image: /assets/article_images/storida-strategy/success-strategy.svg
image2: /assets/article_images/storida-strategy/success-strategy.svg
---

## Storybooks Are Not Made Monthly

Monthly subscription models have a premise: users return every month. AI storybook generation does not fit this premise. Storybooks are made for birthdays, anniversaries, special moments — 2-3 times per year.

The transition: subscription to per-story payment. Simultaneously, a one-tap mode was designed to lower the creation barrier.

## One-Tap "Today's Story" — Start with a Name

The existing creation flow had many settings. Style, prompt, characters, scene count, character count. UX simplification reduced these to 3 essentials, but an even more extreme path was needed: a mode where entering a name is all it takes.

The one-tap mode design principle: **what the user does not decide, the server decides.**

```
User input: child's name (1 field)
Server decides:
  - Situation → random selection from template pool
  - Writing style → random selection
  - Art style → random selection
  - Scene count: 3 (fixed)
  - Chars per scene: 150 (fixed)
```

Situation templates are stored as a JSON array in the system settings table. Admins can add and modify them without code deployment. Templates contain `{{name}}` placeholders. The server substitutes them and passes the result to the existing generation pipeline unchanged.

Frontend changes are minimal. One `mode` parameter is added to the existing generation API. When `mode: 'quick'`, the server fills the rest. The client sends only the name.

```
Creation page top section:
┌─────────────────────────────────────┐
│ "Tell your child a story tonight"   │
│                                     │
│ [Child's name]  [Today's story ▶]  │
│                                     │
│ ↓ Scroll down for detailed settings │
└─────────────────────────────────────┘
```

Detailed settings remain intact. One-tap mode is an entry point, not a replacement of existing features.

## Per-Story Payment — Single Confirmation, Not Billing Key

The existing subscription payment uses billing keys. A card is registered, and charges occur automatically each month. Per-story payment is different: the user approves each payment. It is one-time.

With TossPayments, the difference between the two approaches is clear.

| Item | Subscription (Billing Key) | Per-Story (Single Payment) |
|------|---------------------------|---------------------------|
| Card registration | Required (authKey → billingKey) | Not required |
| Payment approval | Server automatic (billingKey) | User explicit |
| API | Billing key issue + billing key charge | confirm (paymentKey + orderId) |
| Recurrence | Monthly automatic | Each time manual |

The existing Toss API call logic from subscription payment was extracted into a shared utility. Billing key issuance, billing key payment, single confirmation, and payment cancellation are unified in one client class. Both subscription service and token purchase service share the same client.

## Token Package Design

The per-story payment unit is a "token package." A product table stores prices and token amounts set by admins.

Example product configuration:

| Package | Tokens | Price | Meaning |
|---------|--------|-------|---------|
| 1 story | 750 | ₩3,900 | Base unit |
| 3 stories | 2,250 | ₩9,900 | Bundle discount |
| 10 stories | 7,500 | ₩29,900 | Volume discount |

The product list is served via a public API. No authentication required — unauthenticated users should be able to check pricing first.

## Payment Flow

Payment splits into 3 stages.

**Stage 1 — Checkout**: The user selects a product, and the server creates an order. The server returns an order ID, amount, and payment client key.

**Stage 2 — Payment Widget**: The Toss payment widget appears. The user enters card information and approves payment. On success, a `paymentKey` is returned.

**Stage 3 — Confirm**: The client sends the `paymentKey` to the server. The server calls the Toss confirmation API. On success, tokens are credited immediately.

```
User → select package → POST checkout → receive orderId
     → Toss widget payment → receive paymentKey
     → POST confirm → tokens credited
```

The existing token system's `addon_tokens` field is reused. Token history records with `source: 'addon_purchase'`. Since it is managed separately from subscription tokens, the deduction order follows existing logic (add-on first → subscription tokens).

## Code Reuse Strategy

A new feature, but no new patterns.

- **Payment**: Toss client extracted as shared utility. Subscription and per-story both use it.
- **Tokens**: `addon_tokens` and `token_history` used as-is. Only one new table: the product list.
- **Generation**: `mode` parameter added. Rest of the pipeline unchanged.
- **Payment widget**: Toss SDK already used for subscription payment, reused here.

The principle: add features following existing patterns, no architecture changes.

## Conclusion

Storybook creation frequency does not match subscription models. Per-story payment aligns with user behavior patterns. One-tap mode eliminates "settings fatigue." Combined, the entry barrier reaches its minimum. Enter name → pay → storybook complete. Three steps.
