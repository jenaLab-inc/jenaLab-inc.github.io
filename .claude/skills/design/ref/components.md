# Component Styling (Part 6)

> Reference file for `design/SKILL.md` — See main skill for core philosophy and AX system.

## 6.1 Cards

```
Feel: paper card on a table / small menu / booklet page
Background: warm neutral (not pure white)
Border: subtle tone difference (not strong grey line)
Shadow: small and soft — rgba(47,42,39,0.06-0.10)
Content padding: enough breathing room between text and image

Good elements inside cards:
  - Small line illustration
  - Thin divider
  - Micro thumbnail top banner
  - Corner decorative line (one, subtle)
```

**AX-based variants:**
```css
/* AX-JP */  border-radius: 4px; border: 1px solid var(--border-soft); no shadow.
/* AX-MU */  border-radius: 16px; border: 2px solid var(--mucha-gold); bg: var(--mucha-cream).
/* AX-AN */  border: none; border-radius: 12px; backdrop-filter: blur(8px); bg: rgba(255,255,255,0.85).
/* AX-BQ */  bg: var(--baroque-navy); color: var(--antique-ivory); border: 1px solid var(--baroque-gold).
```

## 6.2 Buttons

```
Priority: function over decoration
Primary: deep natural color (Deep Herb #586250) or ink (Espresso #241D1A)
Secondary: thin outline or toned-down background
Hover/Pressed: brightness + shadow shift — not aggressive color change
AVOID: strong glow, jelly/bouncy feel, neon borders
```

**Tone recommendations:**
```
Natural style:    Moss Green / Deep Herb
Interior style:   Cafe Wood / Espresso
Vintage style:    Dusty Clay / Soft Brass
Night mode:       warm neutral background + Brass Glow highlight
```

**AX-based variants:**
```css
/* AX-JP/MN */ border: 1px solid var(--border-soft); background: transparent;
/* AX-MU */    border: 2px solid var(--mucha-gold); border-radius: 24px; font: serif;
/* AX-BQ */    background: var(--baroque-gold); text-transform: uppercase; letter-spacing: 0.08em;
/* AX-SK */    border: 2px solid var(--ink-black); border-radius: 2px; (apply SVG roughness filter)
/* AX-AN */    background: var(--shinkai-blue); box-shadow: 0 0 20px rgba(74,143,202,0.25);
```

## 6.3 Form Inputs

```
Labels: always visible outside the field (never placeholder-as-label)
Placeholder: supplementary hint text only
Input background: very light paper / plaster tone
Focus: clear but not aggressive
Focus ring color: Moss / Brass / Slate — not harsh blue
```

## 6.4 Navigation

```
Role: deliver structure, not decoration
Style: clean, trim alignment
Icons: thin outline line style
Selected state: muted background + text emphasis (not filled color block)
AX-JP/MN: simple horizontal nav, text links only, underline animation on hover (2px, ease-out)
AX-MU: curved border below nav bar, Old Gold on hover
AX-BQ: sticky nav, translucent dark background, gold base divider, serif uppercase logo
AX-SK: hand-drawn style icons, watercolor underline effect on hover
```

## 6.5 Modals / Sheets

```
Feel: small booklet or menu card unfolding
Background: warm paper (Warm Paper #F7F3EC or Candle Cream #F2DEC2)
Handle/header: light grip indicator or thin decorative line
Default: solid surface (warm paper)
AX-AN >= 3: responsible glassmorphism allowed (see 6.7)
```

## 6.6 Status Components (Toast / Banner / Alert)

```
Keep functional clarity; colors should not clash
Success / Warning / Error: use muted tones, not fluorescent
Distinguish states with: luminance contrast + icon — not hue alone
```

## 6.7 Responsible Glassmorphism (2026)

Glassmorphism is permitted as a **functional depth layer** — not a cold, techy effect. In this system it translates as **"looking through a cafe window on a rainy evening."**

```
WHEN to use:
  - AX-AN >= 3 (anime background / Shinkai-style atmosphere)
  - Over rich background imagery, video, or gradient hero sections
  - Floating nav bars, overlay cards, modal sheets on atmospheric pages
  - Dark mode surfaces where warm depth improves readability

WHEN NOT to use:
  - AX-MN >= 4 (minimalism suppresses glass effects)
  - AX-JP >= 4 without AX-AN >= 3 (clean structure prefers solid surfaces)
  - Over flat/plain backgrounds (glass needs something behind it)
  - On form inputs or dense data screens

Warm glass tokens:
  --glass-light:    rgba(247, 243, 236, 0.72);
  --glass-dark:     rgba(36, 29, 26, 0.65);
  --glass-blur:     blur(12px);
  --glass-border:   1px solid rgba(216, 207, 195, 0.35);
  --glass-shadow:   0 4px 24px rgba(47, 42, 39, 0.10);

Rules:
  - Blur: 8-16px. Below 8px looks unfinished; above 16px hides the atmosphere layer
  - Background must be WARM-tinted (paper/cream/espresso) — never neutral grey or cold white
  - Border: always include a faint warm border — glass without edges looks unfinished
  - Text on glass: minimum WCAG AA contrast. Add a subtle inner shadow or tint if needed
  - Max 2 glass layers visible simultaneously per viewport
  - DO NOT combine glassmorphism + heavy texture/grain — choose one depth technique
```

```css
/* Light mode glass card */
.glass-card {
  background: var(--glass-light);
  backdrop-filter: var(--glass-blur);
  -webkit-backdrop-filter: var(--glass-blur);
  border: var(--glass-border);
  border-radius: var(--radius-lg);
  box-shadow: var(--glass-shadow);
}

/* Dark mode glass card */
.glass-card--dark {
  background: var(--glass-dark);
  backdrop-filter: var(--glass-blur);
  -webkit-backdrop-filter: var(--glass-blur);
  border: 1px solid rgba(176, 139, 87, 0.15);
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.20);
}
```

## 6.8 Light Skeuomorphism (2026)

A subtle return to tactile realism — not the heavy leather-and-wood skeuomorphism of 2010, but warm, pressable surfaces with gentle depth cues. In this system it translates as **"a well-made menu card you want to press, flip, or slide."**

```
WHEN to use:
  - Primary action buttons, toggles, sliders — components that benefit from "pressable" affordance
  - Cards in commerce/booking contexts where "pick this up" feeling adds clarity
  - AX-BQ >= 2 (depth and material quality are welcomed)
  - AX-SK >= 2 (handcraft texture already present — skeuomorphism adds dimension)

WHEN NOT to use:
  - AX-MN >= 4 (minimalism suppresses material embellishment — use flat + opacity only)
  - Dense data screens, admin panels, settings pages
  - Over glassmorphism surfaces (choose one depth technique per element)
  - Never on text-only elements or navigation links

Techniques:
  - Soft inner shadow for inset/pressed feel: inset 0 1px 2px rgba(47,42,39,0.08)
  - Gentle top highlight for raised feel: inset 0 1px 0 rgba(255,255,255,0.5)
  - Subtle gradient on surface: 2-3% brightness shift top-to-bottom (warm direction)
  - Border: use surface-toned border, not grey — warm stone-linen or plaster tint
  - Pressed state: invert the shadow direction (top highlight -> bottom, outer shadow -> inset)
  - Transition: 150ms ease-out for press, 250ms ease for release

Rules:
  - Keep shadows WARM — rgba(47,42,39,*) or rgba(112,89,74,*), never cold grey
  - Max depth: 2-3px shadow spread — "resting on a surface" not "floating above"
  - Surface color must remain from the warm palette (paper/plaster/cream) — never cold white or grey
  - DO NOT combine with heavy texture/grain — skeuomorphism provides its own depth
  - Limit to 1-2 skeuomorphic component types per screen — overuse kills the subtlety
```

```css
/* Light skeuomorphic button — warm, pressable */
.btn-tactile {
  background: linear-gradient(180deg, var(--rice-white) 0%, var(--surface) 100%);
  border: 1px solid var(--stone-linen);
  box-shadow:
    0 1px 3px rgba(47, 42, 39, 0.08),
    inset 0 1px 0 rgba(255, 255, 255, 0.6);
  transition: box-shadow 0.15s ease-out, transform 0.15s ease-out;
}

.btn-tactile:hover {
  box-shadow:
    0 2px 6px rgba(47, 42, 39, 0.12),
    inset 0 1px 0 rgba(255, 255, 255, 0.6);
}

.btn-tactile:active {
  transform: scale(0.98);
  box-shadow: inset 0 1px 3px rgba(47, 42, 39, 0.10);
}

/* Light skeuomorphic toggle — grabbable */
.toggle-tactile {
  background: var(--soft-plaster);
  border-radius: var(--radius-full);
  box-shadow: inset 0 1px 3px rgba(47, 42, 39, 0.10);
}

.toggle-tactile__thumb {
  background: linear-gradient(180deg, #fff 0%, var(--rice-white) 100%);
  border: 1px solid var(--stone-linen);
  box-shadow: 0 1px 4px rgba(47, 42, 39, 0.12);
  border-radius: 50%;
}
```
