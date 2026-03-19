# CSS Token Sheet (Part 12)

> Reference file for `design/SKILL.md` — Copy-paste ready CSS custom properties.

```css
:root {
  /* ===== BASE ===== */
  --bg-base:        #FAFAF7;
  --surface:        #F4F1EB;
  --border-soft:    #E2DDD5;
  --text-primary:   #2C2A26;
  --text-secondary: #6B6560;
  --text-muted:     #9E9890;

  /* ===== ATMOSPHERIC PALETTE ===== */
  --warm-paper:     #F7F3EC;
  --rice-white:     #FBF8F2;
  --soft-plaster:   #EEE7DD;
  --stone-linen:    #D8CFC3;
  --cafe-wood:      #70594A;
  --dusty-clay:     #B9785E;
  --rose-terracotta:#C79A89;
  --sand-ochre:     #C8A46A;
  --sage-leaf:      #A8B39C;
  --moss-green:     #73806B;
  --olive-mist:     #B9BEA1;
  --deep-herb:      #586250;
  --fog-blue:       #D7E1E3;
  --rain-blue:      #9EAFB5;
  --slate-teal:     #677983;
  --cloud-grey:     #C9D0D2;
  --amber-glow:     #E0C38A;
  --soft-brass:     #B08B57;
  --candle-cream:   #F2DEC2;
  --ink-charcoal:   #2F2A27;
  --espresso:       #241D1A;
  --smoked-plum:    #6F6270;
  --walnut-shadow:  #4E413A;

  /* ===== AX-SK: Urban Sketch ===== */
  --ink-black:      #1A1A1A;
  --sepia-wash:     #C4A882;
  --bleeding-blue:  #7BA7C2;

  /* ===== AX-JP: Japanese Clarity ===== */
  --ivory-white:    #F8F5F0;
  --sumi:           #3C3C3C;
  --asagi:          #48929B;
  --sakura:         #F0C4C4;

  /* ===== AX-MU: Mucha Ornament ===== */
  --mucha-gold:     #BFA265;
  --mucha-cream:    #F5ECD7;
  --burgundy:       #8C3A4B;

  /* ===== AX-AN: Anime Background ===== */
  --shinkai-blue:   #4A8FCA;
  --ghibli-green:   #88B892;
  --sunset-peach:   #F2B880;
  --magichour-pink: #E8A0B4;
  --cloud-white:    #EEF0F5;

  /* ===== AX-BQ: Baroque Drama ===== */
  --baroque-navy:   #1B1F3B;
  --baroque-gold:   #D4A843;
  --crimson:        #9B2335;
  --antique-ivory:  #F0E6D3;

  /* ===== TYPOGRAPHY ===== */
  --font-body:    'Pretendard', 'Inter', sans-serif;
  --font-display: 'Noto Serif KR', 'Playfair Display', serif;
  --font-accent:  'Italiana', 'Nanum Myeongjo', serif;

  /* ===== SPACING (8px base) ===== */
  --space-1: 4px;   --space-2: 8px;   --space-3: 12px;
  --space-4: 16px;  --space-5: 24px;  --space-6: 32px;
  --space-7: 48px;  --space-8: 64px;  --space-9: 96px;
  --space-10: 128px;

  /* ===== BORDER RADIUS ===== */
  --radius-sm: 4px;   --radius-md: 8px;
  --radius-lg: 16px;  --radius-xl: 24px;
  --radius-full: 9999px;

  /* ===== SHADOWS ===== */
  --shadow-sm:       0 1px 3px rgba(47,42,39,0.06);
  --shadow-md:       0 4px 12px rgba(47,42,39,0.08);
  --shadow-lg:       0 8px 32px rgba(47,42,39,0.12);
  --shadow-glow:     0 0 20px rgba(74,143,202,0.2);
  --shadow-dramatic: 0 12px 40px rgba(27,31,59,0.25);
  --shadow-warm:     0 4px 16px rgba(112,89,74,0.12);

  /* ===== GLASSMORPHISM (2026 — activate when AX-AN >= 3) ===== */
  --glass-light:     rgba(247, 243, 236, 0.72);
  --glass-dark:      rgba(36, 29, 26, 0.65);
  --glass-blur:      blur(12px);
  --glass-border:    1px solid rgba(216, 207, 195, 0.35);
  --glass-shadow:    0 4px 24px rgba(47, 42, 39, 0.10);
}

/* ===== DARK MODE ===== */
@media (prefers-color-scheme: dark) {
  :root {
    --bg-base:        #201B19;
    --surface:        #2A2320;
    --border-soft:    #3D3530;
    --text-primary:   #F3EBDD;
    --text-secondary: #C9BFB1;
    --text-muted:     #8D8278;
  }
}
```
