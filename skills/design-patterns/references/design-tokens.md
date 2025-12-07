# Design Tokens

## W3C Design Tokens Standard (2025)

The W3C Design Tokens Community Group published the first stable specification in October 2025, defining how tokens should be structured, named, and exchanged between tools.

### Token Format (DTCG)

```json
{
  "color": {
    "brand": {
      "primary": {
        "$type": "color",
        "$value": "oklch(55% 0.25 250)",
        "$description": "Primary brand color"
      }
    }
  },
  "spacing": {
    "small": {
      "$type": "dimension",
      "$value": "8px"
    }
  }
}
```

---

## Token Architecture

### 3-Tier Token System

```
┌─────────────────────────────────────────────────┐
│  PRIMITIVE TOKENS (Global/Core)                 │
│  Raw values - the actual colors, sizes, fonts   │
│                                                 │
│  --blue-50, --blue-100, ... --blue-900         │
│  --spacing-1, --spacing-2, ... --spacing-12    │
│  --font-sans, --font-mono                      │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│  SEMANTIC TOKENS (Alias/Purpose)                │
│  What the value means - intent-based naming     │
│                                                 │
│  --color-primary → var(--blue-500)             │
│  --color-background → var(--gray-50)           │
│  --color-text → var(--gray-900)                │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│  COMPONENT TOKENS (Specific)                    │
│  Where the value is used - component-scoped     │
│                                                 │
│  --button-bg → var(--color-primary)            │
│  --card-border → var(--color-border)           │
│  --input-focus-ring → var(--color-primary)     │
└─────────────────────────────────────────────────┘
```

### Why 3 Tiers?

| Tier | Change Frequency | Scope |
|------|------------------|-------|
| Primitive | Rarely (rebrand) | Global |
| Semantic | Sometimes (theme) | Feature |
| Component | Often (UI tweaks) | Component |

---

## CSS Custom Properties

### Defining Tokens

```css
:root {
  /* ═══════════════════════════════════════════
     PRIMITIVE TOKENS
     ═══════════════════════════════════════════ */

  /* Colors - OKLCH for perceptual uniformity */
  --gray-50: oklch(98% 0 0);
  --gray-100: oklch(96% 0 0);
  --gray-200: oklch(92% 0 0);
  --gray-300: oklch(87% 0 0);
  --gray-400: oklch(70% 0 0);
  --gray-500: oklch(55% 0 0);
  --gray-600: oklch(45% 0 0);
  --gray-700: oklch(35% 0 0);
  --gray-800: oklch(25% 0 0);
  --gray-900: oklch(15% 0 0);
  --gray-950: oklch(10% 0 0);

  --blue-50: oklch(97% 0.02 250);
  --blue-100: oklch(94% 0.05 250);
  --blue-200: oklch(88% 0.1 250);
  --blue-300: oklch(78% 0.15 250);
  --blue-400: oklch(68% 0.2 250);
  --blue-500: oklch(55% 0.25 250);
  --blue-600: oklch(48% 0.22 250);
  --blue-700: oklch(40% 0.18 250);
  --blue-800: oklch(33% 0.14 250);
  --blue-900: oklch(27% 0.1 250);

  /* Spacing (4px base unit) */
  --spacing-px: 1px;
  --spacing-0: 0;
  --spacing-0-5: 0.125rem;  /* 2px */
  --spacing-1: 0.25rem;     /* 4px */
  --spacing-1-5: 0.375rem;  /* 6px */
  --spacing-2: 0.5rem;      /* 8px */
  --spacing-2-5: 0.625rem;  /* 10px */
  --spacing-3: 0.75rem;     /* 12px */
  --spacing-3-5: 0.875rem;  /* 14px */
  --spacing-4: 1rem;        /* 16px */
  --spacing-5: 1.25rem;     /* 20px */
  --spacing-6: 1.5rem;      /* 24px */
  --spacing-7: 1.75rem;     /* 28px */
  --spacing-8: 2rem;        /* 32px */
  --spacing-9: 2.25rem;     /* 36px */
  --spacing-10: 2.5rem;     /* 40px */
  --spacing-12: 3rem;       /* 48px */
  --spacing-14: 3.5rem;     /* 56px */
  --spacing-16: 4rem;       /* 64px */

  /* Typography */
  --font-sans: "Inter Variable", ui-sans-serif, system-ui, sans-serif;
  --font-mono: "JetBrains Mono", ui-monospace, monospace;

  --text-xs: 0.75rem;       /* 12px */
  --text-sm: 0.875rem;      /* 14px */
  --text-base: 1rem;        /* 16px */
  --text-lg: 1.125rem;      /* 18px */
  --text-xl: 1.25rem;       /* 20px */
  --text-2xl: 1.5rem;       /* 24px */
  --text-3xl: 1.875rem;     /* 30px */
  --text-4xl: 2.25rem;      /* 36px */

  --leading-none: 1;
  --leading-tight: 1.25;
  --leading-snug: 1.375;
  --leading-normal: 1.5;
  --leading-relaxed: 1.625;
  --leading-loose: 2;

  --tracking-tighter: -0.05em;
  --tracking-tight: -0.025em;
  --tracking-normal: 0;
  --tracking-wide: 0.025em;
  --tracking-wider: 0.05em;

  /* Border Radius */
  --radius-none: 0;
  --radius-sm: 0.25rem;     /* 4px */
  --radius-md: 0.375rem;    /* 6px */
  --radius-lg: 0.5rem;      /* 8px */
  --radius-xl: 0.75rem;     /* 12px */
  --radius-2xl: 1rem;       /* 16px */
  --radius-3xl: 1.5rem;     /* 24px */
  --radius-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px 0 oklch(0% 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px oklch(0% 0 0 / 0.1), 0 2px 4px -2px oklch(0% 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px oklch(0% 0 0 / 0.1), 0 4px 6px -4px oklch(0% 0 0 / 0.1);
  --shadow-xl: 0 20px 25px -5px oklch(0% 0 0 / 0.1), 0 8px 10px -6px oklch(0% 0 0 / 0.1);

  /* Transitions */
  --duration-75: 75ms;
  --duration-100: 100ms;
  --duration-150: 150ms;
  --duration-200: 200ms;
  --duration-300: 300ms;
  --duration-500: 500ms;

  --ease-linear: linear;
  --ease-in: cubic-bezier(0.4, 0, 1, 1);
  --ease-out: cubic-bezier(0, 0, 0.2, 1);
  --ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);

  /* Z-Index */
  --z-0: 0;
  --z-10: 10;
  --z-20: 20;
  --z-30: 30;
  --z-40: 40;
  --z-50: 50;

  /* ═══════════════════════════════════════════
     SEMANTIC TOKENS
     ═══════════════════════════════════════════ */

  /* Backgrounds */
  --color-background: var(--gray-50);
  --color-foreground: var(--gray-900);
  --color-card: white;
  --color-card-foreground: var(--gray-900);
  --color-popover: white;
  --color-popover-foreground: var(--gray-900);

  /* Primary */
  --color-primary: var(--blue-500);
  --color-primary-foreground: white;
  --color-primary-hover: var(--blue-600);

  /* Secondary */
  --color-secondary: var(--gray-100);
  --color-secondary-foreground: var(--gray-900);

  /* Muted */
  --color-muted: var(--gray-100);
  --color-muted-foreground: var(--gray-500);

  /* Accent */
  --color-accent: var(--gray-100);
  --color-accent-foreground: var(--gray-900);

  /* Destructive */
  --color-destructive: oklch(55% 0.25 25);
  --color-destructive-foreground: white;

  /* Border & Ring */
  --color-border: var(--gray-200);
  --color-input: var(--gray-200);
  --color-ring: var(--blue-500);

  /* ═══════════════════════════════════════════
     COMPONENT TOKENS
     ═══════════════════════════════════════════ */

  /* Button */
  --button-height-sm: 2.25rem;   /* 36px */
  --button-height-md: 2.5rem;    /* 40px */
  --button-height-lg: 2.75rem;   /* 44px */
  --button-radius: var(--radius-md);
  --button-font-size: var(--text-sm);
  --button-font-weight: 500;

  /* Input */
  --input-height: 2.5rem;        /* 40px */
  --input-radius: var(--radius-md);
  --input-border-width: 1px;
  --input-font-size: var(--text-sm);

  /* Card */
  --card-radius: var(--radius-lg);
  --card-padding: var(--spacing-6);
  --card-shadow: var(--shadow-sm);
}
```

---

## Tailwind CSS 4 Tokens

### @theme Configuration

```css
/* app.css */
@import "tailwindcss";

@theme {
  /* Override or extend the default theme */

  /* Colors */
  --color-brand: oklch(55% 0.25 250);
  --color-brand-light: oklch(75% 0.15 250);
  --color-brand-dark: oklch(35% 0.2 250);

  /* Custom semantic colors */
  --color-success: oklch(65% 0.2 145);
  --color-warning: oklch(75% 0.15 85);
  --color-error: oklch(55% 0.25 25);
  --color-info: oklch(60% 0.15 250);

  /* Typography */
  --font-display: "Cal Sans", var(--font-sans);

  /* Custom spacing */
  --spacing-18: 4.5rem;
  --spacing-22: 5.5rem;

  /* Custom radius */
  --radius-button: 0.5rem;
  --radius-card: 0.75rem;
  --radius-input: 0.375rem;
}
```

### Using Custom Theme Values

```html
<!-- Use directly in classes -->
<button class="bg-brand text-white rounded-button px-4 py-2">
  Click me
</button>

<div class="bg-success/10 text-success p-4 rounded-card">
  Success message
</div>
```

---

## Token Synchronization

### Figma Variables → CSS

```json
// tokens.json (exported from Figma/Tokens Studio)
{
  "colors": {
    "primary": {
      "50": { "$value": "oklch(97% 0.02 250)" },
      "500": { "$value": "oklch(55% 0.25 250)" },
      "900": { "$value": "oklch(27% 0.1 250)" }
    }
  }
}
```

### Style Dictionary Transform

```javascript
// style-dictionary.config.js
export default {
  source: ["tokens/**/*.json"],
  platforms: {
    css: {
      transformGroup: "css",
      buildPath: "src/styles/",
      files: [{
        destination: "tokens.css",
        format: "css/variables",
        options: {
          outputReferences: true
        }
      }]
    },
    tailwind: {
      transformGroup: "js",
      buildPath: "src/styles/",
      files: [{
        destination: "tailwind-tokens.js",
        format: "javascript/es6"
      }]
    }
  }
};
```

---

## Multi-Brand Token System

### Brand-Specific Overrides

```css
/* Base tokens (shared) */
:root {
  --spacing-4: 1rem;
  --radius-md: 0.5rem;
  --font-sans: system-ui, sans-serif;
}

/* Brand A */
[data-brand="brand-a"] {
  --color-primary: oklch(55% 0.25 250);  /* Blue */
  --color-secondary: oklch(65% 0.15 200);
  --font-display: "Inter", var(--font-sans);
}

/* Brand B */
[data-brand="brand-b"] {
  --color-primary: oklch(55% 0.2 145);   /* Green */
  --color-secondary: oklch(70% 0.12 180);
  --font-display: "Poppins", var(--font-sans);
}
```

### Usage

```tsx
function App({ brand }) {
  return (
    <div data-brand={brand}>
      <button className="bg-primary text-white">
        Brand-specific button
      </button>
    </div>
  );
}
```

---

## Token Naming Conventions

### Recommended Pattern

```
--{category}-{property}-{variant}-{state}

Examples:
--color-text-primary
--color-text-secondary
--color-text-disabled
--color-bg-default
--color-bg-hover
--color-border-default
--color-border-focus
--spacing-component-padding
--font-size-heading-lg
--font-weight-body
```

### Avoid

```css
/* ❌ Bad - too specific */
--blue-button-background
--header-title-font-size

/* ❌ Bad - unclear purpose */
--color-1
--size-a

/* ✅ Good - semantic and scalable */
--color-primary
--text-heading-lg
```

---

## Best Practices

### 1. Always Use Tokens

```css
/* ❌ Never hardcode */
.button {
  background: #3b82f6;
  padding: 16px;
}

/* ✅ Use tokens */
.button {
  background: var(--color-primary);
  padding: var(--spacing-4);
}
```

### 2. Fallback Values

```css
.element {
  /* Provide fallback for older browsers */
  background: var(--color-primary, #3b82f6);
}
```

### 3. Computed Values

```css
:root {
  --base-size: 1rem;
  --scale-ratio: 1.25;

  /* Compute from base */
  --text-sm: calc(var(--base-size) / var(--scale-ratio));
  --text-lg: calc(var(--base-size) * var(--scale-ratio));
  --text-xl: calc(var(--text-lg) * var(--scale-ratio));
}
```

### 4. Token Documentation

```css
:root {
  /**
   * @token color-primary
   * @description Main brand color for CTAs and links
   * @usage Buttons, links, focus rings
   * @a11y Requires white text for contrast
   */
  --color-primary: oklch(55% 0.25 250);
}
```
