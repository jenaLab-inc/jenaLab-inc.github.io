# Typography System (Part 4)

> Reference file for `design/SKILL.md` — See main skill for core philosophy and AX system.

## 4.1 Font Selection

| Usage | Korean (recommended) | English (recommended) | Axis |
|-------|----------------------|-----------------------|------|
| Body | Pretendard, Noto Sans KR | Inter, DM Sans | AX-JP, AX-MN |
| Display/Brand | Maruburii, Noto Serif KR | Playfair Display, Cormorant | AX-BQ, AX-MU |
| Accent / Logo | Nanum Myeongjo | Italiana, Poiret One | AX-MU, AX-SK |
| Handcraft mood | Cafe24 Danjunghae | Caveat, Kalam | AX-SK |

**Rules:**
- Maximum 2 typeface families per project
- Functional/data-heavy screens: 1 typeface, weight variation only
- No excessive condensed, black-weight, or calligraphic fonts
- No over-decorated display fonts in form screens

## 4.2 Type Scale

```
--text-xs:    0.75rem  / 12px  — captions, labels
--text-sm:    0.875rem / 14px  — supporting text
--text-base:  1rem     / 16px  — body (default 15px on mobile)
--text-lg:    1.125rem / 18px  — emphasized body
--text-xl:    1.25rem  / 20px  — sub-heading
--text-2xl:   1.5rem   / 24px  — section title
--text-3xl:   1.875rem / 30px  — page title
--text-4xl:   2.25rem  / 36px  — hero headline
--text-5xl:   3rem     / 48px  — landing hero (AX-BQ >= 3 only)
```

## 4.3 Typography Rules

```
Headings:         quiet but clear — use whitespace and line-height for refinement, not size
Body:             generous line-height (1.7-1.8), short paragraphs
Weight hierarchy: prefer weight variation over color emphasis
Numbers/data:     generous surrounding whitespace; charts use 1 main accent + grey layers
AX-JP active:     letter-spacing 0.04-0.08em, strict baseline grid
AX-MN active:     single typeface, no decorative fonts, no text-5xl
AX-BQ active:     serif display, uppercase EN titles, font-weight 700-900, letter-spacing 0.1em
AX-SK active:     letter-spacing -0.02em on headings, watercolor highlight instead of underline
```

## 4.4 Fluid Typography (2026)

Use CSS `clamp()` and variable fonts for type that breathes with the viewport.

```css
--text-base:  clamp(0.9375rem, 0.875rem + 0.25vw, 1rem);
--text-xl:    clamp(1.125rem, 1rem + 0.5vw, 1.25rem);
--text-3xl:   clamp(1.5rem, 1.25rem + 1vw, 1.875rem);
--text-4xl:   clamp(1.75rem, 1.25rem + 2vw, 2.25rem);

/* Variable font weight for scroll-responsive headings (optional, AX-BQ >= 2) */
/* Weight shifts subtly as user scrolls into view: 400 -> 600 */
```

```
Rules:
  - Fluid sizing is the DEFAULT for all text — fixed px/rem only for UI labels and badges
  - DO NOT let fluid scaling create jumps; transitions must feel continuous
  - Variable font weight transitions: ease-out, 0.3-0.5s — match the "unhurried" motion principle
  - AX-MN >= 4: fluid sizing allowed, but variable weight animation suppressed
```

## 4.5 Exaggerated Hierarchy (2026)

Dramatic scale contrast between mega display type and tiny supporting text — the typographic equivalent of a whisper after a shout. Used for editorial impact on landing pages and hero sections.

```
WHEN to use:
  - Hero / opening sections of landing pages
  - Editorial or lookbook layouts
  - AX-BQ >= 2 (dramatic staging welcomed)
  - Gallery / exhibition contexts where typography IS the visual element
  - Section transitions where a large number or word anchors the visual rhythm

WHEN NOT to use:
  - Functional screens (forms, settings, dashboards, checkout)
  - AX-MN >= 4 without AX-BQ >= 2 (pure minimalism avoids dramatic scale)
  - More than 2 mega-scale elements per page — overuse dilutes impact
  - Body content areas — exaggeration belongs to structural landmarks only

Technique:
  - Pair --text-mega (4-10rem fluid) or --text-display (2.5-5.5rem fluid) with --text-xs (0.625-0.75rem)
  - The gap between display and body text should skip at least 2 scale steps
  - Use font-weight 200-300 for mega type (light = elegant), 400-500 for tiny labels (medium = legible)
  - Large type: tight letter-spacing (-0.02 to -0.04em) for visual density
  - Tiny labels: wide letter-spacing (0.1-0.25em) + uppercase for contrast in voice
  - Stacked line breaks in mega type create vertical rhythm as a graphic element

Forbidden:
  - DO NOT use font-weight 700+ on mega-scale type — it overwhelms the page
  - DO NOT combine exaggerated hierarchy with heavy decoration or ornament
  - DO NOT use more than one mega-scale element in a single viewport
```
