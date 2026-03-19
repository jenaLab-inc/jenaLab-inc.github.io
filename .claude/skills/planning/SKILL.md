---
name: planning
description: >
  Apply structured planning, requirements gathering, UX flow design, and documentation standards.
  Use this skill whenever you write PRDs, user stories, acceptance criteria, screen specs,
  wireflows, or any product/project planning artifacts.
---

# Planning & Requirements Skill

## Overview

Good planning prevents scope creep, design thrash, and wasted implementation. This skill defines the standards for writing requirements, designing UX flows, and structuring all planning artifacts. It applies equally to web, mobile web, Android, and iOS projects.

**Core Principle:** Requirements must start from the **user problem and expected outcome**, not from a feature name or technical solution.

---

## 1. Documentation Hierarchy

All planning work follows a 5-level document hierarchy:

### Level 1 — Request Brief
The starting point for any feature or project. Must include:
- User problem (not a feature name)
- Expected outcome
- Urgency level
- Target platform(s)
- Constraints

### Level 2 — PRD / Feature Spec
**Mandatory fields:**
1. Background
2. Problem statement
3. Goals / KPIs (measurable)
4. Target users & scenarios
5. In-scope features
6. Out-of-scope (explicitly listed)
7. Functional requirements
8. Non-functional requirements (performance, accessibility, security)
9. Dependencies
10. Release criteria

> PRDs answer "what and why", not "how". Avoid over-specifying implementation.

### Level 3 — Journey Map + User Flow
- **Journey Map**: The full user journey from awareness to goal completion — includes emotions, context, pain points, touchpoints
- **User Flow**: Specific step-by-step task completion path within the product
- Large-scale projects need both. Small features may need only a user flow.

### Level 4 — Wireflow / Screen Spec
Never design screens in isolation. Always in the format:
**entry condition → screen → state changes → next screen**

Every screen spec must include:
- Screen ID and name
- Purpose of the screen
- Entry conditions
- User goal
- Primary content
- Primary CTA (max 1 per screen)
- Secondary actions
- Input elements
- **All states**: default / loading / empty / error / no permission / offline / partial success
- Success → next screen
- Failure → recovery path
- Analytics events
- API / data dependencies
- Accessibility notes

### Level 5 — Acceptance Criteria / QA Checklist
Every requirement must have a testable "done" condition.

**User Story format:**
```
As a [user type],
I want [action],
so that [expected benefit].
```

**Acceptance criteria format:**
```
Given [precondition]
When [user action]
Then [expected result]

+ Error case:
+ Accessibility requirement:
+ Analytics event:
```

---

## 2. Requirements Writing Rules

1. **Write from the problem, not the feature.**
   - ❌ "Build a notification center."
   - ✅ "Enable users to rediscover missed critical status changes within 24 hours to reduce churn."

2. **One requirement = one unit of user behavior.** Do not bundle search + save + share + payment into one item.

3. **Always declare both scope and out-of-scope.** Without explicit exclusions, requirements inflate.

4. **Separate functional from non-functional requirements.**
   - Functional: "What it does"
   - Non-functional: "How fast, reliably, securely, accessibly"

5. **No requirement is complete without a success metric.** Minimum one measurable outcome: conversion rate, completion rate, return rate, error rate, average processing time.

6. **State design is part of the requirement.** Always include: loading, empty, error, offline, permission denied, partial success, retry.

7. **Include a review screen before any transaction.** For payments, legal consent, account changes, or destructive actions — a "Check answers / Review" screen is mandatory before the confirmation step.

---

## 3. Definition of Ready

**Do not begin design or development without all 8 of the following:**

```
[ ] User problem and target user defined
[ ] Goal / KPI defined
[ ] Scope and out-of-scope documented
[ ] Main user flow sketched
[ ] Screen list enumerated
[ ] All error/exception states defined
[ ] Acceptance criteria written
[ ] Analytics / tracking events defined
```

---

## 4. UX Flow Design Guide

### The 7-Stage Flow Model

Every feature, regardless of platform, should be designed through these stages:

| Stage | Question it Answers |
|-------|---------------------|
| 1. Entry | Where does the user land? How do they arrive? |
| 2. Orientation | Where am I? What can I do here? |
| 3. Task | How do I complete the core action with minimum steps? |
| 4. Review | Can I verify my input before committing? |
| 5. Commit | Submit / save / pay / confirm |
| 6. Feedback | Did it succeed? Did it fail? What happened? |
| 7. Recovery / Re-entry | Can I undo, edit, retry, or move to the next action? |

> Journey map = entire journey view. User flow = one specific task through these stages.

### Screen Composition Rules

1. **One primary CTA per screen.** Save, submit, and delete should never share the same visual weight. One primary action, multiple secondary actions.

2. **Top of screen shows current context.** Title, current step, back navigation, and progress indicator if applicable.

3. **Content order: Read → Decide → Act.** Information comes first, action comes after. Especially for payments, deletions, or irreversible sends.

4. **Empty state is a temporary state.** Do not place important persistent information only in empty states — they may disappear.

5. **Modals are for short, focused tasks only.** Modals block parent screen interaction. Never use modals for complex flows or forms. Prefer sheets (iOS) or bottom sheets (Android) for contextual tasks.

### Form / Transaction UX Rules

1. **Labels must always be visible.** Never use placeholder text as the sole label for an input field.

2. **Error messages must be text-based.** Red border alone is insufficient. Show an error summary at the top of the page AND an inline error next to the field.

3. **Preserve user input on errors.** Never clear what the user typed when validation fails.

4. **Error messages must be specific.**
   - ❌ "An error occurred."
   - ✅ "Email format is invalid. Example: name@example.com"

---

## 5. Platform-Specific UX Standards

### Web
- Fully keyboard navigable. Focus states must be clearly visible.
- Minimum touch target: 24×24 CSS px (WCAG 2.2 AA floor; aim for 44×44 in touch-heavy UIs)
- Visible label text must match the accessible name (for voice input compatibility)
- All form fields require labels or clear instructions with data format examples
- Hover/focus-triggered temporary content (tooltips, popovers) must be predictable — not flash or block interaction

### Mobile Web
- Assume a "no hover" environment by default. Do not hide key actions behind hover.
- Account for browser chrome, system bars, software keyboard, and safe areas when positioning sticky CTAs or inputs.
- Long forms must be multi-step. Include a review screen before final submission.

### Android
- Design for compact / medium / expanded window size classes simultaneously.
- Navigation pattern by screen size:
  - Compact → Navigation Bar (3–5 top destinations)
  - Medium → Navigation Rail (3–7 destinations)
  - Expanded / deep hierarchy → Navigation Drawer (5+ or 2+ levels)
- Tabs are for in-screen content categorization, not top-level section navigation.
- Minimum touch target: 48dp
- Do not hide key actions behind gestures alone — always provide a visible alternative
- Design for edge-to-edge. Account for status/navigation bar insets, keyboard, cutout, safe zone.

### iOS
- Top-level section navigation → Tab Bar
- Do not use Segmented Control for switching between fully different app sections (use Tab Bar)
- Sheets are for short, contextually close tasks. Users expect swipe-down to dismiss.
- Alerts are for critical, time-sensitive information only. Use sheets/action sheets/menus for choices.
- Minimum hit area: 44×44pt for all interactive elements
- Primary action → Full-width button preferred
- Custom buttons must show a pressed/highlighted state

---

## 6. Templates

### 6.1 Request Brief Template
```
[Request Name]
- Background:
- User Problem:
- Target Users:
- Current Pain / Risk:
- Expected Outcome:
- Success Metric (KPI):
- Target Platform(s): Web / Mobile Web / Android / iOS
- Priority:
- Deadline:
- Constraints:
- Reference Materials:
```

### 6.2 PRD Template
```
[Feature PRD]
1. Purpose
2. Problem Statement
3. Target Users / Use Scenarios
4. Scope (In)
5. Out of Scope
6. Functional Requirements
7. Non-Functional Requirements
8. Information Architecture / User Journey
9. Key User Flows
10. Screen List
11. State Definitions (loading / empty / error / offline / permission)
12. Data / API / Analytics Events
13. Acceptance Criteria
14. QA Checkpoints
15. Release / Rollback Criteria
```

### 6.3 Screen Spec Template
```
[SCREEN-ID]
- Screen Name:
- Purpose:
- Entry Condition:
- User Goal:
- Primary Content:
- Primary CTA:
- Secondary Actions:
- Input Elements:
- States:
    default | loading | empty | error | no permission | offline
- On Success → Next Screen:
- On Failure → Recovery:
- Analytics Events:
- Accessibility Notes:
- API / Data Dependencies:
```

### 6.4 User Story + Acceptance Criteria Template
```
[User Story]
As a [user type],
I want [action],
so that [expected benefit].

[Acceptance Criteria]
Scenario 1 — Happy Path:
  Given ...
  When ...
  Then ...

Scenario 2 — Error Case:
  Given ...
  When ...
  Then ...

Scenario 3 — Edge Case:
  ...

Non-functional:
  - Performance: ...
  - Accessibility: ...
  - Analytics event fired: ...
```

---

## 7. One-Line Planning Rules Summary

- Requirements start with **user problem + expected outcome**, not a feature name.
- Design in order: journey map → user flow → wireflow → screen spec.
- Every screen has: **default / loading / empty / error / offline / permission denied** states.
- Every form needs: **visible labels, descriptive errors, preserved input, and a review step**.
- Definition of Ready: all 8 boxes must be checked before any design or dev work starts.
- No requirement is complete without a **measurable success metric**.
