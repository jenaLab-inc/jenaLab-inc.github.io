---
layout: post
ref: storida-token-pricing
title: "Token-Based Pricing — Designing an AI SaaS Billing Model"
date: 2026-03-04 10:00:00
categories: business
author: jenalab
image: /assets/article_images/storida/token-pricing.svg
image2: /assets/article_images/storida/token-pricing.svg
---

## The Problem: AI Costs Vary

A 3-scene book and a 20-scene book differ by 7x in AI API cost. "N books per month" fails as a pricing unit because "one book" has no fixed cost. Users overpay or the service loses money.

Token-based billing solves this.

## Token Unit Design

### Relationship to AI API Tokens

```
Storida Token = AI API tokens consumed / 10
```

Dividing by 10 produces user-friendly numbers. Users see hundreds, not thousands.

### Tokens Per Scene

| Component | AI Tokens | Storida Tokens |
|---|---|---|
| Image generation (Gemini) | ~1,800 | ~180 |
| Text generation (Claude) | ~500-700 | ~50-70 |
| **Per scene total** | **~2,500** | **~250** |

Working backward from this formula:

| Scene Count | Token Cost |
|---|---|
| 3 scenes (minimum) | 750 |
| 5 scenes | 1,250 |
| 10 scenes | 2,500 |
| 20 scenes | 5,000 |
| 30 scenes (maximum) | 7,500 |

Simple multiplication. Users predict their cost before generating.

## Pricing Plans

| | Start | Basic | Standard | Artist |
|---|---|---|---|---|
| **Price** | Free | 9,900 KRW/mo | 19,900 KRW/mo | 39,900 KRW/mo |
| **Monthly tokens** | 2,500 | 15,000 | 40,000 | 120,000 |
| **Max scenes** | 3 (fixed) | 10 | 20 | 30 |
| **Books at 3 scenes** | ~3 | ~20 | ~53 | ~160 |
| **Books at 10 scenes** | -- | ~6 | ~16 | ~48 |

### Design Principles

1. **Start = trial**: 2,500 tokens covers three 3-scene books (750 x 3 = 2,250) with margin. Scene count locked at 3 to prevent excessive token burn.
2. **Basic = entry paid**: Up to 10 scenes. 6 books worth. Custom characters unlocked.
3. **Standard = core tier**: 20 scenes. PDF download. Print-on-demand eligible.
4. **Artist = professional**: 30 scenes. Custom writing styles unlocked.

Each tier gates features, not just token volume. Higher plans unlock capabilities that justify the price jump.

## Token Management Policies

### No Carryover

Subscription tokens reset on each billing cycle. Unused tokens expire.

On subscription renewal, the user's remaining tokens are reset to the plan's monthly allocation, and the reset timestamp advances to the next billing period.

Carryover allows token hoarding. A user accumulates 6 months of tokens, then burns them in one burst. AI costs spike unpredictably. Monthly reset keeps usage forecasts stable.

### Add-on Tokens: Never Expire

When subscription tokens run out, users buy add-on tokens separately. These never expire.

The user record maintains two separate token balances: subscription tokens (reset monthly) and add-on tokens (permanent). Two separate fields. Two separate lifecycles. No mixing.

### Deduction Order

On generation, add-on tokens deplete first. Any remaining shortfall comes from subscription tokens.

Add-on tokens deplete first. User expectation: "tokens I paid for separately get used first." Subscription tokens reset monthly anyway. Preserving subscription tokens and spending add-on tokens benefits the user.

## DB Schema

### credit_logs: Full Audit Trail

Each credit log entry records the user, transaction type (charge or usage), source event, signed amount, and a snapshot of both token balances at the time of the transaction. Optional references link to the associated book or purchase record, and a human-readable description completes the entry.

| Source Event | Type | Trigger |
|---|---|---|
| Subscription reset | charge | Monthly billing cycle |
| Subscription signup | charge | New subscription |
| Add-on purchase | charge | One-time token purchase |
| Book generation | usage | Book generation |
| Admin adjustment | charge/usage | Manual admin correction |

### Balance Snapshot Design

Each log entry captures both token balances at the moment of the transaction. Three benefits:

1. **Time-series balance lookup**: Check balance at any point with a single query.
2. **Inconsistency detection**: Continuity gaps in `balance_after` reveal missed or duplicate transactions.
3. **Audit support**: Customer service resolves disputes with exact transaction history.

Without snapshots, reconstructing historical balances requires replaying all transactions from the beginning. Snapshots trade storage for query simplicity.

## Business Logic: Generation Eligibility Check

```
// Token validation before generation:
// 1. Calculate required tokens: 250 * page count
// 2. Sum available tokens: subscription + add-on balances
// 3. If insufficient, return the exact shortfall amount
```

On insufficient tokens, the client receives the exact deficit. The purchase prompt displays: "You need 1,250 more tokens. Buy add-on tokens?" Precision drives conversion.

## Pre-Deduction Strategy

Tokens are deducted at request time, not at completion.

```
Request:    deduct tokens -> register job -> respond immediately
On failure: refund tokens (credit_logs records the refund)
```

The alternative — deduct after completion — opens a 2-4 minute window between request and completion. During that window, another request can spend the same tokens. Double-spending.

Pre-deduction eliminates this race condition. The trade-off: refund logic on failure. Refunds are simpler than concurrent deduction bugs.

### Refund Flow

```
Generation fails
  -> Scheduler marks job FAILED
  -> Refund tokens to user
  -> Audit log records the refund with a balance snapshot
```

Every refund is auditable. The token history shows deduction, failure, and refund as separate entries.

## Summary

250 tokens per scene. 3 scenes cost 750. 10 scenes cost 2,500. The formula is transparent. Users calculate their own costs. The service predicts its own expenses. Both sides benefit from the same simple math.

Next: Cloudflare R2 storage migration and Scheduler modularization.
