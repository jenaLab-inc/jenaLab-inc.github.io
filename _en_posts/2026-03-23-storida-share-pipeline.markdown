---
layout: post
ref: storida-share-pipeline
title: "Social Sharing Pipeline — From SSR Share Pages to a Viral Loop"
date: 2026-03-23 10:00:00
categories: architecture
author: jenalab
lang: en
image: /assets/article_images/storida-strategy/share-pipeline.svg
image2: /assets/article_images/storida-strategy/share-pipeline.svg
---

## Gallery Exists, Users Do Not

The AI storybook generation pipeline is complete. A public gallery is built. One thing is missing: a structure where results bring the next user. We designed a social sharing pipeline.

Implementation splits into 4 tracks: share page (SSR), Kakao SDK integration, preset chain, and referral coupons. Dependency analysis maximizes parallel work.

## 1. SSR Share Page — Dynamic OG Design

When someone receives a share link, what they see in Kakao is the OG preview. This preview decides the click. Static meta tags cannot work — each storybook has a different title, cover, and description.

Next.js App Router Server Components solve this.

```
On /book/[id] access:
1. Server-side fetch of storybook data (SSR)
2. Private or not found → 404
3. generateMetadata() creates dynamic OG tags
4. opengraph-image handler generates 1200×630 image
5. Caching: revalidate 3600 (1 hour)
```

The OG image composites cover image + title + brand logo. No separate image server — Next.js `ImageResponse` generates it dynamically.

The key insight: the share page is not just a viewer. It is a **conversion device**. A "Create with this style" CTA sits at the bottom. This button is where the viral loop begins.

## 2. Kakao SDK — 3-Tier Fallback Strategy

The primary sharing channel for Korean users is KakaoTalk. Kakao SDK's `sendDefault` implements feed-type sharing.

The share button has a 3-tier fallback structure.

| Priority | Method | Condition |
|----------|--------|-----------|
| 1 | Kakao SDK | `window.Kakao` exists |
| 2 | Web Share API | `navigator.share` exists (mobile) |
| 3 | Clipboard copy | Everything else (desktop) |

Kakao SDK loads with the `lazyOnload` strategy. It does not block page load. Initialization happens once after load completion.

The share button component is designed for reuse. It accepts `title`, `description`, `imageUrl`, and `shareUrl` as props. The same component works on the share page, gallery preview, and card overlay.

## 3. Preset Chain — From Share to Creation

Clicking the CTA on the share page navigates to the creation page. The original storybook's style and prompt are passed as URL parameters.

```
/create?style_id=X&writing_prompt_id=Y
```

The creation page's state management hook receives these parameters. Priority has 3 tiers:

```
URL preset > localStorage previous settings > system defaults
```

When a URL preset exists, the page opens with that style and prompt pre-selected. The user only needs to enter a name. Friction minimized.

The advantage of this pattern: the change scope is small. The creation page itself is not modified. One `initialPreset` argument is added to the state initialization logic. Later, the gallery's "Create with this style" button uses the same URL pattern.

## 4. Referral Coupon — Incentivizing Shares

Sharing alone is insufficient. Recipients need an incentive.

The coupon system has two phases.

**Generation**: When a user clicks the share button, the server issues a unique code. This code is appended to the share URL as a `ref` parameter.

**Redemption**: When a recipient visits the share link, the `ref` value is saved to localStorage. After signup, the coupon is automatically applied and free tokens are granted.

Four validation rules:

- Expiration check (30 days from issuance)
- Unused status check
- Self-referral blocked
- (Optional) Per-user redemption limit

## 5. Dependency-Based Implementation Order

Analyzing dependencies across the 4 tracks maximizes parallelism.

```
Week 1 ──────────────────────────────
├─ Track A (independent): 3 UX improvements
├─ Track B (sequential): Share page → preset → Kakao button
│
Week 2 ──────────────────────────────
├─ Track C (independent): Per-story payment + one-tap
│
Week 3~4 ────────────────────────────
├─ Track D (after B): Clone feature + coupons
```

Tracks A and B are fully parallel. Track C can also run in parallel but starting after Week 1 is more stable. Only Track D waits for B's completion — the clone feature requires the share page and preset chain.

## Conclusion

Social sharing is not a single feature. SSR page, dynamic OG, Kakao SDK, preset chain, and coupon system connect as one pipeline. Each part works independently, but the viral loop only completes when all pieces are assembled. Results that bring the next user — that is the purpose of this design.
