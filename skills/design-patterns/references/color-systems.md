# Color Systems

## OKLCH Color Space

OKLCH is the recommended color space for modern CSS (2025+), offering perceptual uniformity and P3 wide-gamut support.

### Syntax

```css
/* oklch(Lightness Chroma Hue / Alpha) */
color: oklch(55% 0.25 250);
color: oklch(55% 0.25 250 / 0.5);  /* 50% opacity */

/* L: 0-100% (perceived lightness) */
/* C: 0-0.4+ (saturation/vividness) */
/* H: 0-360 (hue angle) */
```

### Why OKLCH Over HSL/RGB?

| Feature | HSL | OKLCH |
|---------|-----|-------|
| Perceptual uniformity | ❌ | ✅ |
| P3 wide-gamut | ❌ | ✅ |
| Consistent lightness | ❌ | ✅ |
| Palette generation | Hard | Easy |
| Browser support (2025) | 100% | 92%+ |

### The Problem with HSL

```css
/* HSL lightness is NOT perceptually uniform */
/* Both are L: 50% but blue looks darker than yellow */
--yellow: hsl(60 100% 50%);  /* Looks bright */
--blue: hsl(240 100% 50%);   /* Looks dark */

/* OKLCH fixes this */
--yellow: oklch(90% 0.2 100);  /* Adjusted for perception */
--blue: oklch(55% 0.25 250);   /* Same perceived lightness */
```

---

## Color Palette Generation

### Automatic Scale from Base Color

```css
:root {
  /* Define base color */
  --brand-hue: 250;
  --brand-chroma: 0.25;

  /* Generate scale by adjusting lightness */
  --brand-50:  oklch(97% 0.02 var(--brand-hue));
  --brand-100: oklch(94% 0.05 var(--brand-hue));
  --brand-200: oklch(88% 0.1 var(--brand-hue));
  --brand-300: oklch(78% 0.15 var(--brand-hue));
  --brand-400: oklch(68% 0.2 var(--brand-hue));
  --brand-500: oklch(55% var(--brand-chroma) var(--brand-hue));
  --brand-600: oklch(48% 0.22 var(--brand-hue));
  --brand-700: oklch(40% 0.18 var(--brand-hue));
  --brand-800: oklch(33% 0.14 var(--brand-hue));
  --brand-900: oklch(27% 0.1 var(--brand-hue));
  --brand-950: oklch(20% 0.08 var(--brand-hue));
}
```

### Common Hue Values

| Color | Hue (H) | Notes |
|-------|---------|-------|
| Red | 25 | Warm red |
| Orange | 50 | Vibrant orange |
| Yellow | 90 | Golden yellow |
| Lime | 130 | Bright green |
| Green | 145 | Natural green |
| Teal | 180 | Blue-green |
| Cyan | 200 | Light blue |
| Blue | 250 | Primary blue |
| Indigo | 280 | Deep purple-blue |
| Purple | 300 | Violet |
| Pink | 350 | Magenta-pink |

---

## Complete Color System

### Full Implementation

```css
:root {
  /* ═══════════════════════════════════════════
     GRAY SCALE (Neutral)
     ═══════════════════════════════════════════ */
  --gray-50:  oklch(98.5% 0 0);
  --gray-100: oklch(96% 0 0);
  --gray-200: oklch(92% 0 0);
  --gray-300: oklch(87% 0 0);
  --gray-400: oklch(70% 0 0);
  --gray-500: oklch(55% 0 0);
  --gray-600: oklch(45% 0 0);
  --gray-700: oklch(37% 0 0);
  --gray-800: oklch(27% 0 0);
  --gray-900: oklch(18% 0 0);
  --gray-950: oklch(12% 0 0);

  /* ═══════════════════════════════════════════
     PRIMARY (Blue)
     ═══════════════════════════════════════════ */
  --primary-50:  oklch(97% 0.015 250);
  --primary-100: oklch(94% 0.04 250);
  --primary-200: oklch(88% 0.08 250);
  --primary-300: oklch(78% 0.14 250);
  --primary-400: oklch(66% 0.19 250);
  --primary-500: oklch(55% 0.24 250);
  --primary-600: oklch(47% 0.22 250);
  --primary-700: oklch(40% 0.19 250);
  --primary-800: oklch(34% 0.15 250);
  --primary-900: oklch(28% 0.11 250);
  --primary-950: oklch(22% 0.08 250);

  /* ═══════════════════════════════════════════
     SUCCESS (Green)
     ═══════════════════════════════════════════ */
  --success-50:  oklch(97% 0.015 145);
  --success-100: oklch(94% 0.04 145);
  --success-200: oklch(88% 0.08 145);
  --success-300: oklch(78% 0.14 145);
  --success-400: oklch(68% 0.18 145);
  --success-500: oklch(60% 0.2 145);
  --success-600: oklch(52% 0.18 145);
  --success-700: oklch(44% 0.15 145);
  --success-800: oklch(36% 0.12 145);
  --success-900: oklch(30% 0.09 145);
  --success-950: oklch(22% 0.06 145);

  /* ═══════════════════════════════════════════
     WARNING (Amber)
     ═══════════════════════════════════════════ */
  --warning-50:  oklch(98% 0.02 85);
  --warning-100: oklch(95% 0.05 85);
  --warning-200: oklch(90% 0.1 85);
  --warning-300: oklch(84% 0.14 85);
  --warning-400: oklch(78% 0.16 85);
  --warning-500: oklch(75% 0.15 85);
  --warning-600: oklch(65% 0.14 85);
  --warning-700: oklch(52% 0.12 60);
  --warning-800: oklch(44% 0.1 55);
  --warning-900: oklch(38% 0.08 50);
  --warning-950: oklch(28% 0.06 50);

  /* ═══════════════════════════════════════════
     ERROR (Red)
     ═══════════════════════════════════════════ */
  --error-50:  oklch(97% 0.015 25);
  --error-100: oklch(94% 0.04 25);
  --error-200: oklch(88% 0.08 25);
  --error-300: oklch(78% 0.14 25);
  --error-400: oklch(66% 0.2 25);
  --error-500: oklch(55% 0.24 25);
  --error-600: oklch(47% 0.22 25);
  --error-700: oklch(40% 0.19 25);
  --error-800: oklch(34% 0.15 25);
  --error-900: oklch(28% 0.11 25);
  --error-950: oklch(20% 0.08 25);

  /* ═══════════════════════════════════════════
     INFO (Cyan)
     ═══════════════════════════════════════════ */
  --info-50:  oklch(97% 0.015 200);
  --info-100: oklch(94% 0.04 200);
  --info-200: oklch(88% 0.08 200);
  --info-300: oklch(78% 0.12 200);
  --info-400: oklch(68% 0.14 200);
  --info-500: oklch(60% 0.15 200);
  --info-600: oklch(52% 0.14 200);
  --info-700: oklch(44% 0.12 200);
  --info-800: oklch(36% 0.1 200);
  --info-900: oklch(30% 0.08 200);
  --info-950: oklch(22% 0.06 200);
}
```

---

## P3 Wide Gamut

### What is P3?

Display P3 has 50% more colors than sRGB. Modern Apple devices, newer monitors, and phones support it.

### P3-Safe Implementation

```css
:root {
  /* sRGB fallback */
  --vibrant-blue: oklch(55% 0.25 250);

  /* P3 enhancement for supported displays */
  @media (color-gamut: p3) {
    --vibrant-blue: oklch(55% 0.32 250);  /* Higher chroma */
  }
}
```

### Gamut Checking

```css
/* Check if P3 is supported */
@supports (color: oklch(50% 0.3 250)) {
  .element {
    background: oklch(55% 0.3 250);
  }
}

/* Media query approach */
@media (color-gamut: p3) {
  .element {
    background: oklch(55% 0.32 250);
  }
}
```

---

## Color Manipulation

### Relative Color Syntax

```css
:root {
  --primary: oklch(55% 0.25 250);

  /* Darken by reducing lightness */
  --primary-dark: oklch(from var(--primary) calc(l - 0.15) c h);

  /* Lighten by increasing lightness */
  --primary-light: oklch(from var(--primary) calc(l + 0.2) c h);

  /* Desaturate by reducing chroma */
  --primary-muted: oklch(from var(--primary) l calc(c - 0.1) h);

  /* Shift hue */
  --primary-shifted: oklch(from var(--primary) l c calc(h + 30));

  /* Add transparency */
  --primary-transparent: oklch(from var(--primary) l c h / 0.5);
}
```

### color-mix() Function

```css
:root {
  --primary: oklch(55% 0.25 250);
  --white: oklch(100% 0 0);
  --black: oklch(0% 0 0);

  /* Mix colors */
  --primary-light: color-mix(in oklch, var(--primary), var(--white) 30%);
  --primary-dark: color-mix(in oklch, var(--primary), var(--black) 30%);

  /* Create hover state */
  --primary-hover: color-mix(in oklch, var(--primary), var(--black) 15%);

  /* Create disabled state */
  --primary-disabled: color-mix(in oklch, var(--primary), var(--gray-400) 50%);
}
```

---

## Accessibility & Contrast

### WCAG Requirements

| Level | Text Contrast | UI Contrast |
|-------|---------------|-------------|
| AA (minimum) | 4.5:1 | 3:1 |
| AAA (enhanced) | 7:1 | 4.5:1 |
| Large text (18px+) | 3:1 | 3:1 |

### OKLCH Lightness for Contrast

```css
/*
 * Rule of thumb for OKLCH contrast:
 * - Lightness difference of ~40-50% usually meets AA
 * - Dark text (L: 15-25%) on light bg (L: 95-100%)
 * - Light text (L: 95-100%) on dark bg (L: 15-35%)
 */

:root {
  /* Light mode - high contrast */
  --text-primary: oklch(15% 0 0);     /* L: 15% */
  --bg-primary: oklch(100% 0 0);      /* L: 100% */
  /* Difference: 85% ✅ */

  /* Dark mode - high contrast */
  --text-primary-dark: oklch(95% 0 0);  /* L: 95% */
  --bg-primary-dark: oklch(12% 0 0);    /* L: 12% */
  /* Difference: 83% ✅ */
}

/* For colored backgrounds */
.button-primary {
  /* Primary blue: L: 55% */
  background: var(--primary-500);
  /* White text: L: 100%, difference: 45% ✅ */
  color: white;
}
```

### Testing Contrast

```javascript
// Use a contrast checker library
import { getContrast } from 'color2k';

const contrast = getContrast('#ffffff', '#3b82f6');
console.log(`Contrast ratio: ${contrast}:1`);
// Output: Contrast ratio: 4.68:1 (passes AA)
```

---

## Tailwind CSS 4 Colors

### Custom Colors in @theme

```css
@import "tailwindcss";

@theme {
  /* Override default colors */
  --color-blue-500: oklch(55% 0.25 250);
  --color-blue-600: oklch(47% 0.22 250);

  /* Add custom colors */
  --color-brand: oklch(55% 0.25 250);
  --color-brand-light: oklch(75% 0.15 250);
  --color-brand-dark: oklch(35% 0.2 250);

  /* Semantic colors */
  --color-success: oklch(60% 0.2 145);
  --color-warning: oklch(75% 0.15 85);
  --color-error: oklch(55% 0.24 25);
}
```

### Usage

```html
<button class="bg-brand hover:bg-brand-dark text-white">
  Brand Button
</button>

<div class="bg-success/10 text-success border border-success/20">
  Success alert
</div>
```

---

## Semantic Color Mapping

### Light Mode

```css
:root {
  /* Surface colors */
  --color-background: var(--gray-50);
  --color-surface: white;
  --color-surface-raised: white;

  /* Text colors */
  --color-text-primary: var(--gray-900);
  --color-text-secondary: var(--gray-600);
  --color-text-tertiary: var(--gray-500);
  --color-text-disabled: var(--gray-400);

  /* Border colors */
  --color-border-default: var(--gray-200);
  --color-border-muted: var(--gray-100);
  --color-border-emphasis: var(--gray-300);

  /* Interactive */
  --color-interactive: var(--primary-500);
  --color-interactive-hover: var(--primary-600);
  --color-interactive-active: var(--primary-700);
}
```

### Dark Mode

```css
.dark {
  /* Surface colors */
  --color-background: var(--gray-950);
  --color-surface: var(--gray-900);
  --color-surface-raised: var(--gray-800);

  /* Text colors */
  --color-text-primary: var(--gray-50);
  --color-text-secondary: var(--gray-400);
  --color-text-tertiary: var(--gray-500);
  --color-text-disabled: var(--gray-600);

  /* Border colors */
  --color-border-default: var(--gray-800);
  --color-border-muted: var(--gray-900);
  --color-border-emphasis: var(--gray-700);

  /* Interactive - slightly brighter in dark mode */
  --color-interactive: var(--primary-400);
  --color-interactive-hover: var(--primary-300);
  --color-interactive-active: var(--primary-500);
}
```

---

## Tools & Resources

### Color Pickers

- **OKLCH Picker**: [oklch.com](https://oklch.com)
- **OKLCH.fyi**: [oklch.fyi](https://oklch.fyi)

### Palette Generators

- **Huetone**: Contrast-safe palette generation
- **Leonardo**: Adobe's contrast-based color tool
- **Realtime Colors**: Live preview on UI

### Contrast Checkers

- **WebAIM Contrast Checker**
- **Polypane Color Contrast**
- Browser DevTools (Chrome, Firefox)
