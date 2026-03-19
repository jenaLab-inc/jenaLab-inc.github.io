# Motion and Interaction (Part 8)

> Reference file for `design/SKILL.md` — See main skill for core philosophy and AX system.

## 8.1 Motion Character

> The movement of this style is not "spectacular" — it is "breath."

```
Pace:     unhurried
Quality:  non-intrusive, doesn't pop
Metaphors: wind / light / paper / curtain / water surface
```

## 8.2 Recommended Motion Types

```
Fade in/out
Short slide (directional, not diagonal)
Gentle sheet-up
Subtle scale change (1.0 -> 1.02)
Slow background focus
Hover: brightness shift + shadow micro-change (not strong color jump)
```

## 8.2.1 Motion Posters & Ambient Loops (2026)

Subtle, looping background motion adds life without demanding attention — like watching light shift through a window.

```
Allowed ambient motion:
  - Hero background video (muted, low-saturation, overlaid with warm tint) — max 1 per page
  - Slow CSS gradient drift on hero/banner backgrounds (60-120s cycle)
  - Gentle particle/light-mote float on landing hero (AX-AN >= 3 only)
  - Looping illustration element: swaying leaf, flickering candle, drifting cloud

Rules:
  - Ambient loops MUST pause when prefers-reduced-motion is set
  - Video backgrounds: desaturate (saturate 0.4-0.6) and overlay warm tint — never raw footage
  - Motion must be peripheral; if the user watches the motion instead of reading content, it's too much
  - Max 1 ambient motion source per viewport at any time
  - DO NOT autoplay audio. Ever.
  - Performance: lazy-load video, provide poster/fallback image

DO NOT use as ambient motion:
  - Pulsing/breathing UI elements
  - Auto-scrolling text/carousels
  - Looping transition effects on functional components
```

## 8.2.2 Functional Micro-Interactions (2026)

Small, purposeful feedback animations that confirm user actions and guide attention — each one should feel like a gentle nod, not a shout.

```
Recommended micro-interactions:
  - Button press: subtle inward scale (0.97) + shadow reduction, 150ms ease-out
  - Toggle/switch: smooth slide with soft color fade, 200ms
  - Card hover: lift (translateY -2px) + shadow expansion, 300ms ease
  - Form focus: border color transition + faint glow bloom, 250ms
  - Success state: brief brightness pulse on the element, 400ms ease-out
  - Scroll reveal: fade-in + translateY(12-16px), staggered by 60-80ms per element
  - Tab/nav switch: underline width expansion (0->100%), 250ms ease-out

Timing guidelines:
  - Immediate feedback (press/toggle): 100-200ms
  - State transitions (focus/hover): 200-350ms
  - Reveals and entrance: 400-700ms
  - Easing: ease-out for entrances, ease-in-out for state changes — never linear

AX-MN >= 4: reduce to opacity transitions only; suppress scale/translate micro-interactions
```

## 8.2.3 Physical-World Animation (2026)

Make UI elements behave like physical objects — buttons that compress, surfaces that tilt, scrolls that carry momentum. The goal is **tactile believability**, not spectacle.

```
Allowed physical-world techniques:
  - Pressable button: scale(0.97-0.98) + shadow collapse on :active — feels like pushing a real button
  - Reactive card tilt: subtle rotate3d on hover tracking cursor position (max +/-3deg) — AX-BQ >= 2 only
  - Magnetic snap: element gently drifts toward cursor within 60px radius, then snaps on click — hero CTAs only
  - Elastic scroll: content overshoots by 2-4px then settles on scroll-snap — matches iOS native feel
  - Surface response: hover triggers subtle shadow shift matching light direction — "object on table" under moving lamp
  - Toggle slide: thumb moves with spring easing (cubic-bezier(0.34, 1.56, 0.64, 1)), 200ms

Rules:
  - Physical animation is ADDITIVE to existing micro-interactions — not a replacement
  - Max 1 reactive/tracking element per viewport (tilt card OR magnetic CTA, not both)
  - Performance: use transform and opacity only — never animate layout properties
  - Mobile: disable cursor-tracking effects (tilt, magnetic); keep press/spring effects
  - Spring easing is allowed ONLY on small elements (toggles, chips) — never on page-level transitions
  - AX-MN >= 4: suppress all physical-world animation except press feedback
  - AX-JP >= 4: limit to press and shadow-shift — no tilt or magnetic effects

DO NOT:
  - Add physics-based bounce to page scroll or section reveals
  - Make entire cards or containers tilt — limit to hero elements or featured items
  - Use inertia or momentum on horizontal carousels (conflicts with scroll-snap precision)
  - Combine physical animation with glassmorphism on the same element — visual noise
```

## 8.3 Forbidden Motion

```
Strong bounce
Rubber-band / jelly transitions
Excessive parallax
Flame / neon / scanline effects
Decorative elements moving excessively
```

## 8.4 Per-AX Motion

```
AX-SK: hover -> slight tilt (rotate 0.5-1deg) / page transition -> paper-flip slide
AX-JP: hover -> underline expands (width 0->100%) / page -> clean fade (opacity only)
AX-MU: hover -> curved border blooms (SVG path animation) / scroll -> botanical growth sequence
AX-AN: hover -> glow spreads (box-shadow) / background -> slow cloud drift (60s CSS cycle)
AX-BQ: hover -> gold specular sweep / reveal -> opacity + scale 1.02 from darkness
AX-MN >= 4: opacity transition only (all other motion suppressed)
```

## 8.5 State Transition Rules

```
Loading:  quiet — no large spinner or heavy flash
Skeleton: low-contrast paper/wall-like tone (not stark white blocks)
Empty:    small sketch illustration or minimal message with generous whitespace
Success:  brightening feel — not flashing celebration
Error:    noticeable tension without breaking the overall atmosphere
```
