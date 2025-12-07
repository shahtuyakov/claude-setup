# Typography

## Fluid Typography

Fluid typography scales smoothly between viewport sizes using `clamp()`, eliminating the need for breakpoint-based font sizes.

### The clamp() Function

```css
/* clamp(minimum, preferred, maximum) */
font-size: clamp(1rem, 0.5rem + 2vw, 2rem);

/*
 * 1rem = minimum size (won't go smaller)
 * 0.5rem + 2vw = scales with viewport
 * 2rem = maximum size (won't go larger)
 */
```

### Complete Fluid Type Scale

```css
:root {
  /* Fluid type scale - scales from 320px to 1280px viewport */
  --text-xs:   clamp(0.75rem, 0.7rem + 0.25vw, 0.875rem);
  --text-sm:   clamp(0.875rem, 0.8rem + 0.375vw, 1rem);
  --text-base: clamp(1rem, 0.925rem + 0.375vw, 1.125rem);
  --text-lg:   clamp(1.125rem, 1rem + 0.625vw, 1.375rem);
  --text-xl:   clamp(1.25rem, 1rem + 1.25vw, 1.75rem);
  --text-2xl:  clamp(1.5rem, 1rem + 2.5vw, 2.25rem);
  --text-3xl:  clamp(1.875rem, 1rem + 4.375vw, 3rem);
  --text-4xl:  clamp(2.25rem, 1rem + 6.25vw, 4rem);
  --text-5xl:  clamp(3rem, 1rem + 10vw, 6rem);
}
```

### Calculating clamp() Values

```
Formula for preferred value:
preferred = minSize + (maxSize - minSize) * ((100vw - minViewport) / (maxViewport - minViewport))

Simplified:
preferred = minSize(rem) + slope(vw)

Where:
slope = (maxSize - minSize) / (maxViewport - minViewport) * 100
```

### Generator Tool

```javascript
// Fluid type calculator
function fluidType(minSize, maxSize, minVW = 320, maxVW = 1280) {
  const slope = (maxSize - minSize) / (maxVW - minVW);
  const yAxisIntersection = minSize - slope * minVW;

  const slopeVW = (slope * 100).toFixed(4);
  const interceptRem = (yAxisIntersection / 16).toFixed(4);

  return `clamp(${minSize / 16}rem, ${interceptRem}rem + ${slopeVW}vw, ${maxSize / 16}rem)`;
}

// Usage
fluidType(16, 24);  // Base text: 16px → 24px
// Output: clamp(1rem, 0.5rem + 0.8333vw, 1.5rem)
```

---

## Type Scale Ratios

### Common Ratios

| Ratio | Name | Use Case |
|-------|------|----------|
| 1.067 | Minor Second | Minimal contrast |
| 1.125 | Major Second | Body copy, UI |
| 1.200 | Minor Third | General purpose |
| 1.250 | Major Third | Headlines |
| 1.333 | Perfect Fourth | Strong hierarchy |
| 1.414 | Augmented Fourth | Display type |
| 1.500 | Perfect Fifth | Bold statements |
| 1.618 | Golden Ratio | Dramatic contrast |

### Scale Implementation

```css
:root {
  --scale-ratio: 1.25;  /* Major Third */
  --base-size: 1rem;

  /* Calculate scale */
  --text-xs:   calc(var(--base-size) / var(--scale-ratio) / var(--scale-ratio));
  --text-sm:   calc(var(--base-size) / var(--scale-ratio));
  --text-base: var(--base-size);
  --text-lg:   calc(var(--base-size) * var(--scale-ratio));
  --text-xl:   calc(var(--base-size) * var(--scale-ratio) * var(--scale-ratio));
  --text-2xl:  calc(var(--base-size) * var(--scale-ratio) * var(--scale-ratio) * var(--scale-ratio));
}
```

---

## Container Query Typography

### Typography Based on Container Size

```css
/* Define container */
.card {
  container-type: inline-size;
  container-name: card;
}

/* Scale typography with container */
.card-title {
  font-size: clamp(1rem, 3cqi + 0.5rem, 2rem);
}

/* Container query for breakpoints */
@container card (min-width: 400px) {
  .card-title {
    font-size: 1.5rem;
  }
}

@container card (min-width: 600px) {
  .card-title {
    font-size: 2rem;
  }
}
```

### Container Query Units

| Unit | Description |
|------|-------------|
| cqi | 1% of container's inline size |
| cqw | 1% of container's width |
| cqh | 1% of container's height |
| cqmin | Smaller of cqi or cqb |
| cqmax | Larger of cqi or cqb |

### Accessibility Warning

```css
/*
 * ⚠️ Container query units can fail WCAG 1.4.4 (Resize Text)
 * Always include a rem-based minimum in clamp()
 */

/* ❌ May not resize properly with browser zoom */
font-size: 5cqi;

/* ✅ Includes rem base for zoom support */
font-size: clamp(1rem, 1rem + 3cqi, 2rem);
```

---

## Font Stack

### System Font Stack

```css
:root {
  --font-sans: ui-sans-serif, system-ui, sans-serif, "Apple Color Emoji", "Segoe UI Emoji";
  --font-serif: ui-serif, Georgia, Cambria, "Times New Roman", Times, serif;
  --font-mono: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
}
```

### Variable Fonts

```css
/* Load variable font */
@font-face {
  font-family: "Inter Variable";
  src: url("/fonts/Inter-Variable.woff2") format("woff2");
  font-weight: 100 900;
  font-style: normal;
  font-display: swap;
}

:root {
  --font-sans: "Inter Variable", ui-sans-serif, system-ui, sans-serif;
}

/* Use font-variation-settings for fine control */
.heading {
  font-family: var(--font-sans);
  font-weight: 650;  /* Any value between 100-900 */
}
```

### Popular Variable Fonts (2025)

| Font | Use Case | Source |
|------|----------|--------|
| Inter | UI, body text | Google Fonts |
| Geist | Modern UI | Vercel |
| Plus Jakarta Sans | Headlines | Google Fonts |
| JetBrains Mono | Code | Google Fonts |
| Cal Sans | Display | Cal.com |

---

## Line Height & Spacing

### Line Height Scale

```css
:root {
  --leading-none: 1;
  --leading-tight: 1.25;
  --leading-snug: 1.375;
  --leading-normal: 1.5;
  --leading-relaxed: 1.625;
  --leading-loose: 2;
}

/* Usage guidelines */
h1, h2, h3 { line-height: var(--leading-tight); }
p, li { line-height: var(--leading-relaxed); }
.compact { line-height: var(--leading-snug); }
```

### Letter Spacing

```css
:root {
  --tracking-tighter: -0.05em;
  --tracking-tight: -0.025em;
  --tracking-normal: 0;
  --tracking-wide: 0.025em;
  --tracking-wider: 0.05em;
  --tracking-widest: 0.1em;
}

/* Large text often needs negative tracking */
.display-heading {
  font-size: var(--text-5xl);
  letter-spacing: var(--tracking-tight);
}

/* Small caps or uppercase needs positive tracking */
.label {
  text-transform: uppercase;
  letter-spacing: var(--tracking-wider);
}
```

### Paragraph Spacing

```css
/* Optimal line length: 45-75 characters */
.prose {
  max-width: 65ch;
}

/* Paragraph spacing */
.prose p + p {
  margin-top: 1.5em;
}

/* Heading spacing */
.prose h2 {
  margin-top: 2em;
  margin-bottom: 0.75em;
}
```

---

## Text Styles System

### Complete Text Styles

```css
:root {
  /* Display - Hero headlines */
  --text-display-lg: 700 var(--text-5xl) / var(--leading-none) var(--font-sans);
  --text-display-md: 700 var(--text-4xl) / var(--leading-tight) var(--font-sans);
  --text-display-sm: 600 var(--text-3xl) / var(--leading-tight) var(--font-sans);

  /* Heading - Section headers */
  --text-heading-lg: 600 var(--text-2xl) / var(--leading-tight) var(--font-sans);
  --text-heading-md: 600 var(--text-xl) / var(--leading-snug) var(--font-sans);
  --text-heading-sm: 600 var(--text-lg) / var(--leading-snug) var(--font-sans);

  /* Body - Paragraph text */
  --text-body-lg: 400 var(--text-lg) / var(--leading-relaxed) var(--font-sans);
  --text-body-md: 400 var(--text-base) / var(--leading-relaxed) var(--font-sans);
  --text-body-sm: 400 var(--text-sm) / var(--leading-normal) var(--font-sans);

  /* Label - UI elements */
  --text-label-lg: 500 var(--text-base) / var(--leading-none) var(--font-sans);
  --text-label-md: 500 var(--text-sm) / var(--leading-none) var(--font-sans);
  --text-label-sm: 500 var(--text-xs) / var(--leading-none) var(--font-sans);

  /* Code - Monospace */
  --text-code: 400 var(--text-sm) / var(--leading-normal) var(--font-mono);
}

/* Usage */
.hero-title { font: var(--text-display-lg); }
.section-title { font: var(--text-heading-lg); }
.paragraph { font: var(--text-body-md); }
.button { font: var(--text-label-md); }
```

---

## Tailwind Typography

### Tailwind CSS 4 Config

```css
@import "tailwindcss";

@theme {
  /* Custom font families */
  --font-display: "Cal Sans", var(--font-sans);

  /* Override sizes for fluid typography */
  --text-xs: clamp(0.75rem, 0.7rem + 0.25vw, 0.875rem);
  --text-sm: clamp(0.875rem, 0.8rem + 0.375vw, 1rem);
  --text-base: clamp(1rem, 0.925rem + 0.375vw, 1.125rem);
  --text-lg: clamp(1.125rem, 1rem + 0.625vw, 1.375rem);
  --text-xl: clamp(1.25rem, 1rem + 1.25vw, 1.75rem);
  --text-2xl: clamp(1.5rem, 1rem + 2.5vw, 2.25rem);
  --text-3xl: clamp(1.875rem, 1rem + 4.375vw, 3rem);
}
```

### Typography Classes

```html
<!-- Display heading -->
<h1 class="text-4xl md:text-5xl font-bold tracking-tight leading-none">
  Hero Headline
</h1>

<!-- Section heading -->
<h2 class="text-2xl font-semibold tracking-tight">
  Section Title
</h2>

<!-- Body text -->
<p class="text-base text-muted-foreground leading-relaxed max-w-prose">
  Body content with comfortable reading width.
</p>

<!-- Small label -->
<span class="text-xs font-medium uppercase tracking-wider text-muted-foreground">
  Category Label
</span>
```

### @tailwindcss/typography Plugin

```html
<!-- Prose class for rich text content -->
<article class="prose prose-lg dark:prose-invert max-w-none">
  <h1>Article Title</h1>
  <p>First paragraph with automatic styling...</p>
  <h2>Subheading</h2>
  <ul>
    <li>Styled list item</li>
  </ul>
</article>
```

---

## Responsive Typography Patterns

### Breakpoint-Based (Traditional)

```css
h1 {
  font-size: 2rem;
}

@media (min-width: 768px) {
  h1 { font-size: 3rem; }
}

@media (min-width: 1024px) {
  h1 { font-size: 4rem; }
}
```

### Fluid (Recommended)

```css
h1 {
  font-size: clamp(2rem, 1rem + 5vw, 4rem);
}
/* No media queries needed */
```

### Hybrid Approach

```css
h1 {
  /* Fluid scaling within bounds */
  font-size: clamp(2rem, 1rem + 5vw, 4rem);

  /* Adjust other properties at breakpoints */
  letter-spacing: -0.02em;
}

@media (min-width: 1024px) {
  h1 {
    letter-spacing: -0.04em;
  }
}
```

---

## Accessibility

### Minimum Font Sizes

| Element | Minimum Size | Recommended |
|---------|--------------|-------------|
| Body text | 16px | 18px |
| Small text | 14px | 16px |
| Captions | 12px | 14px |
| UI labels | 12px | 14px |

### WCAG Requirements

```css
/*
 * WCAG 1.4.4: Text can be resized to 200% without loss
 * Use rem units, not px
 * Avoid viewport-only units
 */

/* ❌ Fails zoom test */
font-size: 2vw;

/* ✅ Passes zoom test */
font-size: clamp(1rem, 1rem + 1vw, 2rem);
```

### Font Weight for Readability

```css
/* Light text on dark backgrounds needs more weight */
.dark {
  --font-weight-body: 400;  /* Was 400, may need 450+ */
}

/* Small text needs more weight */
.caption {
  font-size: var(--text-sm);
  font-weight: 450;
}
```
