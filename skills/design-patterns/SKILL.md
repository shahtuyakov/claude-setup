---
name: design-patterns
description: Modern design system patterns for 2025. Covers design tokens, OKLCH color systems, fluid typography, animations, dark mode, shadcn/ui components, and Figma handoff. Use when implementing UI styles, theming, accessibility, and visual polish.
---

# Design Patterns

Modern design system patterns optimized for 2025 web development.

## Technology Selection

### CSS Framework

| Option | Best For | Notes |
|--------|----------|-------|
| Tailwind CSS 4 | Most projects | CSS-first config, P3 colors |
| Vanilla CSS | Simple sites | CSS variables, modern features |
| CSS Modules | Component isolation | Scoped styles |

### Component Library

| Option | Best For | Notes |
|--------|----------|-------|
| shadcn/ui | Full control | Copy-paste, Radix + Tailwind |
| Radix UI | Custom design | Headless, accessible primitives |
| Base UI | Future-proof | By Radix team, actively maintained |

### Animation

| Option | Best For | Notes |
|--------|----------|-------|
| Framer Motion | React apps | Declarative, layout animations |
| GSAP | Complex timelines | SVG, scroll-driven |
| CSS transitions | Simple states | No JS, performant |

## Reference Files

| Topic | Load | Use When |
|-------|------|----------|
| Design tokens | `references/design-tokens.md` | Token architecture, CSS variables |
| Color systems | `references/color-systems.md` | OKLCH, palettes, contrast |
| Typography | `references/typography.md` | Fluid type, scales |
| Animation | `references/animation.md` | Framer Motion, micro-interactions |
| Dark mode | `references/dark-mode.md` | Theme switching |
| Components | `references/component-patterns.md` | shadcn/ui, Radix |
| Handoff | `references/design-handoff.md` | Figma Dev Mode |

## Quick Start

### Tailwind CSS 4 Setup

```css
/* app.css */
@import "tailwindcss";

@theme {
  /* Colors - OKLCH for P3 gamut */
  --color-primary: oklch(55% 0.25 250);
  --color-secondary: oklch(65% 0.15 300);
  --color-destructive: oklch(55% 0.25 25);

  /* Typography */
  --font-sans: "Inter Variable", system-ui, sans-serif;
  --font-mono: "JetBrains Mono", monospace;

  /* Spacing (4px base) */
  --spacing-px: 1px;
  --spacing-0: 0;
  --spacing-1: 0.25rem;
  --spacing-2: 0.5rem;
  --spacing-4: 1rem;
  --spacing-8: 2rem;

  /* Border radius */
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --radius-full: 9999px;
}
```

### CSS Variables (Token System)

```css
:root {
  /* Primitives */
  --blue-50: oklch(97% 0.02 250);
  --blue-500: oklch(55% 0.25 250);
  --blue-900: oklch(25% 0.15 250);

  /* Semantic */
  --color-background: var(--blue-50);
  --color-foreground: var(--blue-900);
  --color-primary: var(--blue-500);

  /* Component */
  --button-bg: var(--color-primary);
  --button-text: white;
}

.dark {
  --color-background: oklch(12% 0.02 250);
  --color-foreground: oklch(95% 0.02 250);
}
```

### Fluid Typography

```css
:root {
  /* Fluid type scale */
  --text-xs: clamp(0.75rem, 0.7rem + 0.25vw, 0.875rem);
  --text-sm: clamp(0.875rem, 0.8rem + 0.375vw, 1rem);
  --text-base: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);
  --text-lg: clamp(1.125rem, 1rem + 0.625vw, 1.25rem);
  --text-xl: clamp(1.25rem, 1rem + 1.25vw, 1.5rem);
  --text-2xl: clamp(1.5rem, 1rem + 2.5vw, 2rem);
  --text-3xl: clamp(1.875rem, 1rem + 4.375vw, 3rem);
}
```

### Animation Basics

```tsx
// Framer Motion - Fade in
import { motion } from "framer-motion";

export function FadeIn({ children }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 10 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.3 }}
    >
      {children}
    </motion.div>
  );
}

// Respect reduced motion
const prefersReducedMotion = window.matchMedia(
  "(prefers-reduced-motion: reduce)"
).matches;

const variants = prefersReducedMotion
  ? { initial: {}, animate: {} }
  : { initial: { opacity: 0 }, animate: { opacity: 1 } };
```

### Dark Mode Toggle

```tsx
// Theme provider pattern
import { useEffect, useState } from "react";

export function useTheme() {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  useEffect(() => {
    // Check system preference
    const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;
    const stored = localStorage.getItem("theme");

    setTheme(stored ? stored as "light" | "dark" : prefersDark ? "dark" : "light");
  }, []);

  useEffect(() => {
    document.documentElement.classList.toggle("dark", theme === "dark");
    localStorage.setItem("theme", theme);
  }, [theme]);

  return { theme, setTheme, toggle: () => setTheme(t => t === "light" ? "dark" : "light") };
}
```

## Design Principles

### Token Architecture

```
┌─────────────────────────────────────┐
│  Primitive Tokens                   │
│  Raw values: colors, sizes, fonts   │
│  --blue-500: oklch(55% 0.25 250)   │
├─────────────────────────────────────┤
│  Semantic Tokens                    │
│  Purpose-based: primary, danger     │
│  --color-primary: var(--blue-500)  │
├─────────────────────────────────────┤
│  Component Tokens                   │
│  Specific usage: button-bg          │
│  --button-bg: var(--color-primary) │
└─────────────────────────────────────┘
```

### Accessibility Checklist

- [ ] Color contrast AA (4.5:1 text, 3:1 UI)
- [ ] Focus indicators visible (2px minimum)
- [ ] Touch targets 44x44px minimum
- [ ] Reduced motion support
- [ ] Semantic HTML structure
- [ ] ARIA labels where needed
- [ ] Keyboard navigation works

### Responsive Strategy

```
Mobile-first approach:
1. Base styles = mobile
2. sm: (640px) = large phones
3. md: (768px) = tablets
4. lg: (1024px) = laptops
5. xl: (1280px) = desktops

Container queries for components:
@container (min-width: 400px) { ... }
```

## Common Patterns

### Button with States

```tsx
// Tailwind + shadcn style
<button className="
  inline-flex items-center justify-center
  h-10 px-4 py-2
  bg-primary text-primary-foreground
  rounded-md text-sm font-medium
  transition-colors duration-150
  hover:bg-primary/90
  focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring
  disabled:opacity-50 disabled:pointer-events-none
">
  Click me
</button>
```

### Card with Hover

```tsx
<div className="
  rounded-lg border bg-card p-6
  shadow-sm
  transition-all duration-200
  hover:shadow-md hover:-translate-y-0.5
">
  <h3 className="text-lg font-semibold">Card Title</h3>
  <p className="text-muted-foreground">Card content here.</p>
</div>
```

### Loading Skeleton

```tsx
<div className="animate-pulse">
  <div className="h-4 bg-muted rounded w-3/4 mb-2" />
  <div className="h-4 bg-muted rounded w-1/2" />
</div>
```

## Performance Tips

1. **Use CSS over JS** for simple animations
2. **Avoid layout thrashing** - batch DOM reads/writes
3. **Lazy load** below-fold content
4. **Optimize images** - WebP, proper sizing
5. **Reduce motion** for accessibility & performance
6. **Use `will-change`** sparingly for complex animations
