# Layout System (Part 5)

> Reference file for `design/SKILL.md` — See main skill for core philosophy and AX system.

## 5.1 Grid and Spacing

```
Default grid:   12 columns
Gutter:         24px (mobile: 16px)
Max width:      1200px (AX-MN >= 5 -> 960px)

Mobile spacing units:   16 / 20 / 24
Tablet/Web units:       24 / 32 / 40 / 56
Section gaps:           clearly wider than intra-card gaps

AX-JP whitespace scale:
  0-2 -> default padding (16-24px)
  3   -> padding x 1.5
  4   -> padding x 2
  5   -> padding x 2.5 + min 80px between sections + 30%+ negative space
```

## 5.2 Layout Principles

```
Structure first: alignment must be clean even with atmospheric decoration
Whitespace = atmosphere: generous whitespace IS the mood; never fill for the sake of it
Sketch/texture -> supplementary role only; sketch/watercolor never competes with CTA
Strong hero images -> functional content below must be MORE ordered, not less
```

## 5.3 Per-AX Layout Patterns

```
AX-SK: slight grid breakouts allowed (rotate -1 to 1deg), irregular image masks, z-index layering
AX-JP: strict grid snap, left-align primary, equal gaps throughout, 30%+ negative space
AX-MN: 1 primary action per screen, nav <= 5 items, card content <= 3 lines, outline icons 2px stroke
AX-MU: Art Nouveau SVG borders, corner botanicals, curved dividers, circular hero framing
AX-AN: full-bleed gradient/illustration background, content in centered container
AX-BQ: 100vh hero, dark bg + light text, gold dividers between sections, parallax on scroll
```

## 5.4 Layout by Platform

**Web:** Wider layouts can carry more landscape atmosphere — large images/illustrations, 2-column splits, hero atmospheric staging. Watch: textures covering text, section separation blurring, hero overloading.

**Mobile App:** Information comes first. Distill atmosphere into card headers, sheet surfaces, section tops. Watch: illustration/texture over-density, decoration competing with taps/forms.

**Functional screens** (signup, payment, reservation, settings, data list, admin):
Keep only: warm background tone / clean typography / max 1 thin decorative line / soft shadow / restrained accent color
