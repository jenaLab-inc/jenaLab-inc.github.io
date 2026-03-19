---
name: design
description: >
  Apply the JenaLab Design System when making any visual design decision.
  This system combines two complementary frameworks: the Aesthetic Axis (AX) system for
  high-level style composition, and the Atmospheric Design Guide for detailed visual,
  component, and layout rules. Use this skill for color palettes, typography, layout,
  components, motion, illustration, and responsive design choices.
  Detailed specs are in ref/ files — read them when working on specific domains.
---

# JenaLab Design System — Design Skill

## Core Philosophy

> **"Order as the skeleton. Nature and light as the air. Handcraft and ornament as the fragrance."**

Two complementary frameworks:
1. **Aesthetic Axis (AX) System** — high-level style identity. Six axes, each 0–5 intensity.
2. **Atmospheric Design Guide** — granular rules for color, layout, components, motion, illustration.

**Decision priority (always in this order):**
1. Usability / information structure
2. Ordered layout
3. Light and air quality (atmosphere)
4. Material texture
5. Curved / ornamental decoration
6. Vintage sensibility

Atmosphere and decoration matter — but whenever they conflict with information clarity, they step back.

### 2026 Trend Alignment

| Trend | How It Maps to This System |
|-------|---------------------------|
| **Responsible Glassmorphism** | Warm-toned frosted surfaces as a functional depth layer — not cold glass. Activated via AX-AN ≥ 3 |
| **Fluid Typography** | Variable fonts responding to viewport and scroll, reinforcing the "breathing" quality of layouts |
| **Functional Minimalism** | Already the core of AX-JP + AX-MN; strengthened with intentional micro-interactions |
| **Motion Posters** | Subtle ambient loops (hero video, drifting light) as Layer B atmosphere — never dominant |
| **Nostalgia as Design Tool** | This system's handcraft, vintage, and material warmth IS nostalgia — used deliberately for emotional grounding |
| **Time-Adaptive Theming** | Palette and lighting shift by time-of-day context, extending the Atmospheric Axes |
| **Digital Wellbeing** | Natural pause points, reduced-motion respect, no infinite scroll or aggressive engagement patterns |
| **Light Skeuomorphism** | Warm, pressable surfaces with subtle shadows and soft embossing — "menu card on a table" made tactile. Complements Layer C (Material) |
| **Resonant Stark** | Minimalism with emotional warmth — ultra-thin fonts, soft gradients, generous whitespace. Validates AX-JP:5 + AX-MN:4 approach |
| **Exaggerated Hierarchy** | Dramatic type scale contrast (mega display + tiny labels) for editorial/landing contexts. Controlled via AX-BQ ≥ 2 |
| **Nature Distilled** | Muted earthy tones of skin, wood, soil — this IS the existing Atmospheric Color Library |
| **Human Scribble** | Sketch overlays, doodles, hand-drawn marks for authenticity against AI aesthetics. Maps directly to AX-SK |
| **Physical-World Animation** | Pressable buttons, reactive textures, scrolls that feel alive — extends the "breath" motion principle |

Trends explicitly excluded: Anti-Design (contradicts ordered structure), Zero UI (outside visual scope), aggressive 3D/spatial effects (conflicts with paper/natural texture), Overstimulation/Dial-Up Delight (sensory overload contradicts calm, ordered philosophy), '80s/'90s Excess (opulent maximalism conflicts with restraint), Hyper-Saturated palettes (violates "no neon, no fluorescent" color rule).

---

## Part 1: Aesthetic Axis (AX) System

### 1.1 The Six Aesthetic Axes

| Code | Name | Core Keywords | What It Controls |
|------|------|---------------|-----------------|
| **AX-SK** | Urban Sketch | Pen lines, watercolor bleed, rough texture, handmade feel | Texture & illustration layers |
| **AX-JP** | Japanese Clarity | Order, whitespace, strict grid, restraint | Layout & structure |
| **AX-MN** | Minimalism | Keep only what's essential, functional placement | Information hierarchy & UI density |
| **AX-MU** | Mucha Ornament | Art Nouveau curves, floral motifs, decorative frames | Decoration, typography borders |
| **AX-AN** | Anime Background | Ghibli natural light, Makoto Shinkai skies & glow | Color, gradients, background mood |
| **AX-BQ** | Baroque Drama | Deep chiaroscuro, gold accents, extreme contrast | Emphasis, accent colors, dramatic staging |

### 1.2 Axis Intensity Configuration

At the start of every design project, define the axis intensities (0 = off, 5 = maximum):

```
Project: _______________
Target Users: _______________
One-Line Mood: _______________

AX-SK  Urban Sketch     : _/5
AX-JP  Japanese Clarity  : _/5
AX-MN  Minimalism        : _/5
AX-MU  Mucha Ornament    : _/5
AX-AN  Anime Background  : _/5
AX-BQ  Baroque Drama     : _/5

Primary Axis: ___
Secondary Axis: ___
Accent Axis: ___
```

### 1.3 AX Preset Examples

```
Functional SaaS:    AX-SK:0  AX-JP:5  AX-MN:5  AX-MU:0  AX-AN:2  AX-BQ:1
Children's Story:   AX-SK:4  AX-JP:3  AX-MN:2  AX-MU:3  AX-AN:5  AX-BQ:1
Portfolio Landing:  AX-SK:3  AX-JP:4  AX-MN:3  AX-MU:4  AX-AN:3  AX-BQ:3
Lifestyle / Cafe:   AX-SK:3  AX-JP:4  AX-MN:3  AX-MU:2  AX-AN:2  AX-BQ:1
```

### 1.4 AX Conflict Rules

```
AX-BQ + AX-MN both at intensity ≥ 4 → FORBIDDEN (minimal and dramatic conflict)
AX-SK + AX-JP both at intensity 5   → CAUTION (separate into distinct zones)
Gradients: only when AX-AN ≥ 3. Other axes use flat color.
Gold: AX-MU Old Gold and AX-BQ Gold cannot coexist prominently — assign to different elements.
Dark Mode: triggered when AX-BQ ≥ 3.
AX-MN ≥ 4: textures forbidden, motion minimized, single typeface only.
```

---

## Part 2: Atmospheric Design Guide (Summary)

### Five-Layer Screen Architecture

Every screen is built from five layers in this order (structure first):

```
Layer A — Structure:   grid, whitespace, alignment, type hierarchy, section division
Layer B — Atmosphere:  background tone, light direction, time-of-day air quality
Layer C — Material:    paper grain, wood/linen/ceramic texture, watercolor bleed, brushwork
Layer D — Ornament:    arches, curved frames, thin botanical motifs, decorative lines
Layer E — Motion:      fade, slide, subtle scale, gentle parallax, light bloom effect
```

**Atmosphere and decoration never override Layer A.**

### Style Mixing Ratios (by screen type)

```
Productivity / Admin:       Structure 70 / Atmosphere 15 / Texture 5  / Ornament 5  / Vintage 5
Commerce / Brand:           Structure 55 / Atmosphere 20 / Texture 10 / Ornament 10 / Vintage 5
Content / Story:            Structure 50 / Atmosphere 20 / Texture 15 / Ornament 10 / Vintage 5
Space / Cafe / Hospitality: Structure 45 / Atmosphere 25 / Texture 10 / Ornament 10 / Vintage 10
```

### Six Atmospheric Axes

| Axis | Options |
|------|---------|
| **Space** | Urban Sketch / Natural Landscape / Interior Cafe / Vintage House |
| **Time** | Morning / Overcast Day / Dusk / Rainy Evening / Warm Night |
| **Order** | 1 (mood-first) → 3 (balanced) → 5 (strict minimal) |
| **Texture** | 1 (clean digital) → 3 (slight paper/watercolor) → 5 (visible handcraft) |
| **Ornament** | 1 (none) → 3 (curved lines/frames) → 5 (brand hero areas only) |
| **Temperature** | Cool neutral / Slightly warm / Candlelight cafe mood |

---

## Reference Files Index

Detailed specifications are split into separate files. Read the relevant file when working on that domain:

| File | Contents | When to Read |
|------|----------|-------------|
| [ref/color.md](ref/color.md) | Color tokens, palettes, mixing rules, time-adaptive theming, lighting | Choosing colors, building palettes, dark mode |
| [ref/typography.md](ref/typography.md) | Font selection, type scale, fluid typography, exaggerated hierarchy | Setting up typography, editorial layouts |
| [ref/layout.md](ref/layout.md) | Grid, spacing, per-AX layout patterns, platform rules | Structuring pages, responsive design |
| [ref/components.md](ref/components.md) | Cards, buttons, forms, nav, modals, glassmorphism, light skeuomorphism | Building UI components |
| [ref/illustration.md](ref/illustration.md) | Urban sketch, human scribble, landscape images, motifs, baroque translation | Working with images and decorative elements |
| [ref/motion.md](ref/motion.md) | Motion character, ambient loops, micro-interactions, physical-world animation | Adding animation and interaction |
| [ref/templates.md](ref/templates.md) | Screen type templates, themed project templates (Quiet Sketch, Resonant Stark, etc.) | Starting a new page or project |
| [ref/css-tokens.md](ref/css-tokens.md) | Complete CSS custom properties sheet (copy-paste ready) | Implementing in code |

---

## Project Start Brief Template

Fill this in at the start of every project before designing anything:

```markdown
## Project Style Brief

- Project type:
- Core user emotion:
- Space axis: Urban / Natural / Interior / Vintage
- Time axis: Morning / Day / Dusk / Rain / Night
- Order level (1–5):
- Texture level (1–5):
- Ornament level (1–5):
- Temperature: Neutral / Slightly warm / Candlelight mood
- AX-SK: _/5  AX-JP: _/5  AX-MN: _/5  AX-MU: _/5  AX-AN: _/5  AX-BQ: _/5
- Key scene keyword:
- Atmosphere to avoid:
- Palette template:
- Header style:
- Card style:
- Button style:
- Image/illustration direction:
- Motion tone:
- Dark mode needed:
- Time-adaptive theming: No / Opt-in
- Glassmorphism: No / Light glass / Dark glass
- Light skeuomorphism: No / Buttons only / Buttons + toggles + cards
- Ambient motion (hero video/loop): No / Yes (describe)
- Physical-world animation: No / Press feedback / Tilt + magnetic
- Exaggerated hierarchy: No / Hero only / Hero + section anchors
- Human scribble accents: No / Dividers / Overlays + marks
- Most critical feature on this screen:
```

---

## What to Absolutely Avoid

```
[ ] Decoration overload — atmosphere is fragrance, not the walls
[ ] Prop-heavy vintage feeling
[ ] Excessive vintage noise texture
[ ] Layouts where atmosphere comes before information
[ ] Heavy all-brown palette
[ ] Overly yellow, stuffy indoor tone
[ ] Misaligned screens justified as "natural feel"
[ ] Unreadable dark screens justified as "moody aesthetic"
[ ] Inaccurate UI elements justified as "handcrafted look"
[ ] Same texture/motif pattern repeated on every single screen
[ ] Neon, cyber, fluorescent, or strong saturated accent as primary color
[ ] Text over illustration without a surface layer
[ ] Decoration in primary CTA areas
[ ] Overstimulation / sensory overload layouts (contradicts calm, ordered philosophy)
[ ] Intentionally messy "dial-up" aesthetic or visual noise as decoration
[ ] Hyper-saturated "dopamine" color palettes — warmth ≠ loudness
[ ] Heavy 3D sculptural typography or inflatable visuals (conflicts with paper/natural texture)
[ ] Cold or neutral-grey skeuomorphism — material depth must stay warm-tinted
```

---

## Accessibility Checklist

```
[ ] Text contrast ≥ WCAG AA (4.5:1 body, 3:1 large text)
[ ] Text over textured/illustrated backgrounds → placed on surface card (not directly on texture)
[ ] Buttons look like buttons; links look like links
[ ] Full-card interactive elements have clear hover / focus / pressed states
[ ] Mobile touch targets: sufficiently large with padding
[ ] All decorative elements have aria-hidden="true"
[ ] All animations respect prefers-reduced-motion
[ ] prefers-color-scheme dark mode properly supported
[ ] focus-visible style defined (project accent outline / 2px solid default)
[ ] Font sizes in rem units
```

### Digital Wellbeing (2026)

```
DO:
  - Design natural pause points in long content
  - Provide clear "end of content" signals — no infinite scroll
  - Respect prefers-reduced-motion, prefers-color-scheme, and prefers-contrast

DO NOT:
  - Use infinite scroll without explicit "load more" action
  - Autoplay media with sound
  - Use attention-hijacking patterns (countdown timers, fake urgency badges, pulsing notifications)
  - Design dark patterns that make dismissal harder than engagement
```

---

## Design QA Checklist

Before finalizing any design, most answers to these should be "YES":

**Structure**
- [ ] Is alignment clear and intentional?
- [ ] Is whitespace comfortable and breathing?
- [ ] Does decoration stay out of the information hierarchy?

**Atmosphere**
- [ ] Does the screen feel quiet and comfortable?
- [ ] Is the tone neither too cold nor too garish?

**Material / Texture**
- [ ] Does handcraft texture serve as supporting role (not dominant)?
- [ ] Does "vintage" read as quality — not as worn-out?

**Color**
- [ ] Is text/background contrast sufficient?
- [ ] Are there too many accent colors?

**Components**
- [ ] Do buttons clearly look like buttons?
- [ ] Are functional screens free of excessive atmosphere decoration?
- [ ] Does dark mode feel like a warm night cafe rather than a cold void?

**Interaction**
- [ ] Is motion soft and unhurried?
- [ ] Are hover/focus/pressed states visible but not intrusive?

**2026 Additions**
- [ ] Does glassmorphism (if used) have warm tint and sufficient text contrast?
- [ ] Is typography fluid across breakpoints (clamp-based)?
- [ ] Do ambient motion elements pause with prefers-reduced-motion?
- [ ] Are there natural content endpoints (no infinite scroll)?
- [ ] Does light skeuomorphism (if used) keep shadows warm and depth subtle (max 2–3px)?
- [ ] Is exaggerated hierarchy (if used) limited to max 1 mega-scale element per viewport?
- [ ] Do physical-world animations (tilt, magnetic) disable on mobile?
- [ ] Are human scribble accents (if used) limited to editorial zones, not functional UI?
- [ ] Are no more than 2 depth techniques combined on a single element (glass + skeuo + texture)?
