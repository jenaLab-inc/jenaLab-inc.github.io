# Color System (Part 3)

> Reference file for `design/SKILL.md` — See main skill for core philosophy and AX system.

## 3.1 Base Color Tokens (Always Active)

| Token | Value | Purpose |
|-------|-------|---------|
| `--bg-base` | `#FAFAF7` | Warm white, paper-like canvas |
| `--surface` | `#F4F1EB` | Card & panel background |
| `--border-soft` | `#E2DDD5` | Soft dividers |
| `--text-primary` | `#2C2A26` | Near-ink charcoal |
| `--text-secondary` | `#6B6560` | Supporting text |
| `--text-muted` | `#9E9890` | Hints & captions |

## 3.2 Full Atmospheric Color Library

> **IMPORTANT: No single color family is the default accent.** Choose the primary accent
> family based on the project's identity, audience, and mood. All accent families (B–E)
> are equal candidates. A bakery might lead with warm earth tones, a wellness app with
> cool blues, a garden brand with greens, a luxury brand with brass/gold. Always decide
> per project — never default to one family.

**A. Paper / Wall / Linen (Background family)**
```
Warm Paper:    #F7F3EC  — app canvas, section backgrounds
Rice White:    #FBF8F2  — elevated backgrounds
Soft Plaster:  #EEE7DD  — card surfaces, modals
Stone Linen:   #D8CFC3  — section dividers, muted borders
```

**B. Wood / Earth / Clay (Warm accent family)**
```
Cafe Wood:     #70594A  — secondary accent, cafe-interior feel
Dusty Clay:    #B9785E  — supporting accent, warm CTA variant
Rose Terracotta:#C79A89 — gentle highlight, tag/badge warm version
Sand Ochre:    #C8A46A  — warm decorative accent
```

**C. Garden / Moss / Leaf (Natural accent family)**
```
Sage Leaf:     #A8B39C  — light accent, background support
Moss Green:    #73806B  — natural accent, selected state, positive cues
Olive Mist:    #B9BEA1  — supporting natural tone
Deep Herb:     #586250  — strong natural accent
```

**D. Sky / Rain / Mist (Cool accent family)**
```
Fog Blue:      #D7E1E3  — exterior/rain concept backgrounds
Rain Blue:     #9EAFB5  — calm emphasis, info card backgrounds
Slate Teal:    #677983  — cool accent, supporting text
Cloud Grey:    #C9D0D2  — secondary text, dividers
```

**E. Warm Light / Brass (Glow accent family)**
```
Amber Glow:    #E0C38A  — lighting atmosphere, hover glow
Soft Brass:    #B08B57  — decorative lines, icon accent
Candle Cream:  #F2DEC2  — warmest background variant, modal base
```

**F. Ink / Night / Espresso (Text & dark family)**
```
Ink Charcoal:  #2F2A27  — primary text (replaces pure black)
Espresso:      #241D1A  — dark theme base
Smoked Plum:   #6F6270  — muted supporting text, dark variants
Walnut Shadow: #4E413A  — deep shadow, dark card borders
```

**G. AX-System Extended Tokens (activate by AX intensity)**
```
AX-SK: Ink Black #1A1A1A / Sepia Wash #C4A882 / Bleeding Blue #7BA7C2
AX-JP: Ivory White #F8F5F0 / Sumi #3C3C3C / Asagi #48929B / Sakura #F0C4C4
AX-MU: Mucha Gold #BFA265 / Moss Green #6B7F5E / Burgundy #8C3A4B / Cream #F5ECD7
AX-AN: Shinkai Blue #4A8FCA / Ghibli Green #88B892 / Sunset Peach #F2B880 / Magic Hour Pink #E8A0B4
AX-BQ: Baroque Navy #1B1F3B / Baroque Gold #D4A843 / Crimson #9B2335 / Antique Ivory #F0E6D3
```

## 3.3 Color Mixing Rules

```
Default ratio: 70% neutral background / 20% supporting natural color / 10% accent

Rule 1 — Max 2 strong color groups per screen
Rule 2 — When using natural green + earth brown together: one must be desaturated into a supporting role
Rule 3 — Accent goes to functional elements first (CTA, selection, warning) — not decoration
Rule 4 — Never combine: bright background + high-chroma button + strong texture simultaneously
Rule 5 — No pure white (#FFF) or pure black (#000) as primary backgrounds
Rule 6 — No more than 3 accent colors per screen
Rule 7 — No neon, cyber blue, fluorescent pink, or strong purple as dominant colors
```

## 3.4 Color Selection Algorithm

```
Step 1 — Define project identity:
  What is the brand / product about?
  Who are the target users?
  What emotion should the interface evoke?
  → This determines the PRIMARY ACCENT FAMILY (B, C, D, or E).
  → There is no default. Every project must choose deliberately.

Step 2 — Set the space character (background + supporting tones):
  Interior:    paper + wood + amber
  Exterior:    paper + sky + fog
  Vintage:     plaster + clay + brass
  Rainy:       fog + slate + warm paper
  Night cafe:  espresso + candle cream + brass

Step 3 — Set the time/temperature (adjust warmth):
  Morning/clear:  warm paper + cool or neutral accents
  Golden hour:    plaster + warm accents + amber
  Rain/overcast:  rice white + cool accents
  Night:          espresso + warm neutral + brass

Step 4 — Set function weight:
  Info/settings/productivity:  neutral 75% / accent 5%
  Branding/homepage:           atmosphere 20-25%
  Detail page:                 adjust by image/landscape
  Payment/critical CTA:        contrast and placement over color
```

## 3.5 Ready-Made Palette Templates

> These are **examples** showing how different accent families can lead a design.
> They are NOT defaults. Create a custom palette for each project based on Step 1 of the
> Color Selection Algorithm (project identity).

**Quiet Urban Wash** (cool accent lead — sky/mist family)
```
Canvas: #FBF8F2  |  Surface: #F1EBE2  |  Text: #2F2A27
Secondary Text: #6E655F  |  Main Accent: #9EAFB5
Secondary Accent: #677983  |  Glow: #E0C38A
Best for: note apps, curation, archive, writing apps
```

**Warm Terracotta Studio** (warm accent lead — earth/clay family)
```
Canvas: #F7F3EC  |  Surface: #EEE4D5  |  Text: #312A26
Secondary Text: #7A7069  |  Main Accent: #B9785E
Supporting: #C79A89  |  Glow: #F2DEC2
Best for: lifestyle brand, cafe/restaurant, reservation, slow-content home
```

**Provence Minimal Vintage** (warm glow lead — brass family)
```
Canvas: #F4EDE3  |  Surface: #E8DDCF  |  Text: #3B322D
Secondary Text: #7F7166  |  Main Accent: #B08B57
Supporting: #C8A46A  |  Accent: #C79A89
Best for: living/interior, boutique shop, brand storytelling landing
```

**Garden Morning** (natural accent lead — green family)
```
Canvas: #FBF8F2  |  Surface: #F1EBE2  |  Text: #2F2A27
Secondary Text: #6B6560  |  Main Accent: #73806B
Supporting: #A8B39C  |  Glow: #E0C38A
Best for: wellness, organic brand, garden/plant shop, sustainability
```

**Rainy Filmic Night** (dark mode — cool accent lead)
```
Canvas: #201B19  |  Surface: #2A2320  |  Text: #F3EBDD
Secondary Text: #C9BFB1  |  Main Accent: #8EA0AA
Supporting: #8D7A68  |  Warm Glow: #D8B16D
Best for: dark mode, music/reading/immersive apps, night cafe mood
```

## 3.6 Time-Adaptive Theming (2026)

Interfaces can shift palette and lighting based on the user's time-of-day context.

```
Morning (6:00-11:00):
  Canvas: --rice-white (#FBF8F2) | Accent: project accent (lighter variant) | Glow: none
  Mood: bright, airy, paper-like clarity

Afternoon (11:00-17:00):
  Canvas: --bg-base (#FAFAF7) | Accent: project accent (base variant) | Glow: subtle
  Mood: balanced, neutral warmth

Golden Hour (17:00-20:00):
  Canvas: --warm-paper (#F7F3EC) | Accent: --dusty-clay | Glow: --amber-glow on key elements
  Mood: warm, dusk settling

Night (20:00-6:00):
  Canvas: --espresso (#241D1A) | Accent: --soft-brass | Glow: --candle-cream sparse
  Mood: night cafe, candlelight

Implementation:
  - CSS custom properties swapped by time-aware class on <html>
  - Transition: 1.5-2s ease (daylight changing feel)
  - ALWAYS provide manual override; never force a theme
  - Respect prefers-color-scheme as primary signal
  - Time-adaptation is OPTIONAL and OPT-IN
```

## 3.7 Lighting Rules

```
PREFER:
  - Indirect light over direct / harsh light
  - Soft, spreading brightness over a single sharp highlight
  - Warm grey-brown shadows over black shadows
  - Paper / wood / wall reflections over glass / metal reflections

Shadow style:
  - Low and wide (not deep and dramatic — unless AX-BQ >= 3)
  - Color: warm grey-brown (e.g., rgba(47, 42, 39, 0.08))
  - Card elevation: "menu card on a table" not "floating above the page"

Dark mode:
  - Background: espresso/charcoal/deep plum — never pure #000
  - Text: cream-tinted white — never pure #FFF
  - Accents: brass/amber/fog — never cold neon
```
