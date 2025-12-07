# Dark Mode

## Core Principles

### Don't Invert - Redesign

Dark mode isn't simply inverting colors. Great dark mode requires intentional design decisions.

| Light Mode | Dark Mode | Why |
|------------|-----------|-----|
| Pure white `#FFFFFF` | Dark gray `#121212` | Pure black is harsh |
| Black text `#000000` | Off-white `#E5E5E5` | Pure white causes glare |
| Shadows for depth | Lighter surfaces for depth | Shadows don't work on dark |
| High saturation | Reduced saturation | Vibrant colors strain eyes |

### Recommended Colors

```css
/* ❌ Too harsh */
.dark {
  --background: #000000;
  --text: #ffffff;
}

/* ✅ Comfortable */
.dark {
  --background: oklch(12% 0.01 250);  /* ~#121212 with slight blue */
  --foreground: oklch(93% 0.01 250);  /* Off-white */
}
```

---

## CSS Implementation

### CSS Variables Setup

```css
:root {
  /* Light mode (default) */
  --color-background: oklch(100% 0 0);
  --color-foreground: oklch(15% 0 0);

  --color-card: oklch(100% 0 0);
  --color-card-foreground: oklch(15% 0 0);

  --color-primary: oklch(55% 0.25 250);
  --color-primary-foreground: oklch(100% 0 0);

  --color-secondary: oklch(96% 0.01 250);
  --color-secondary-foreground: oklch(20% 0.02 250);

  --color-muted: oklch(96% 0.01 250);
  --color-muted-foreground: oklch(45% 0.02 250);

  --color-accent: oklch(96% 0.01 250);
  --color-accent-foreground: oklch(20% 0.02 250);

  --color-destructive: oklch(55% 0.25 25);
  --color-destructive-foreground: oklch(100% 0 0);

  --color-border: oklch(90% 0.01 250);
  --color-input: oklch(90% 0.01 250);
  --color-ring: oklch(55% 0.25 250);
}

.dark {
  /* Dark mode overrides */
  --color-background: oklch(12% 0.01 250);
  --color-foreground: oklch(93% 0.01 250);

  --color-card: oklch(15% 0.01 250);
  --color-card-foreground: oklch(93% 0.01 250);

  --color-primary: oklch(60% 0.22 250);  /* Slightly brighter */
  --color-primary-foreground: oklch(10% 0.02 250);

  --color-secondary: oklch(20% 0.02 250);
  --color-secondary-foreground: oklch(93% 0.01 250);

  --color-muted: oklch(20% 0.02 250);
  --color-muted-foreground: oklch(60% 0.02 250);

  --color-accent: oklch(20% 0.02 250);
  --color-accent-foreground: oklch(93% 0.01 250);

  --color-destructive: oklch(60% 0.22 25);  /* Adjusted for dark bg */
  --color-destructive-foreground: oklch(93% 0.01 250);

  --color-border: oklch(22% 0.02 250);
  --color-input: oklch(22% 0.02 250);
  --color-ring: oklch(60% 0.22 250);
}
```

### System Preference Detection

```css
/* Automatic dark mode based on OS setting */
@media (prefers-color-scheme: dark) {
  :root {
    --color-background: oklch(12% 0.01 250);
    --color-foreground: oklch(93% 0.01 250);
    /* ... rest of dark mode variables */
  }
}
```

### Class-Based Toggle

```css
/* Manual toggle with class */
:root {
  /* Light mode variables */
}

.dark {
  /* Dark mode overrides */
}

/* OR use data attribute */
[data-theme="dark"] {
  /* Dark mode overrides */
}
```

---

## React Implementation

### Theme Provider

```tsx
// theme-provider.tsx
"use client";

import { createContext, useContext, useEffect, useState } from "react";

type Theme = "light" | "dark" | "system";

interface ThemeContextType {
  theme: Theme;
  setTheme: (theme: Theme) => void;
  resolvedTheme: "light" | "dark";
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>("system");
  const [resolvedTheme, setResolvedTheme] = useState<"light" | "dark">("light");

  useEffect(() => {
    // Get stored preference
    const stored = localStorage.getItem("theme") as Theme | null;
    if (stored) {
      setTheme(stored);
    }
  }, []);

  useEffect(() => {
    const root = document.documentElement;

    // Determine actual theme
    let actualTheme: "light" | "dark";

    if (theme === "system") {
      actualTheme = window.matchMedia("(prefers-color-scheme: dark)").matches
        ? "dark"
        : "light";
    } else {
      actualTheme = theme;
    }

    // Apply theme
    root.classList.remove("light", "dark");
    root.classList.add(actualTheme);
    setResolvedTheme(actualTheme);

    // Store preference
    localStorage.setItem("theme", theme);
  }, [theme]);

  // Listen for system theme changes
  useEffect(() => {
    if (theme !== "system") return;

    const mediaQuery = window.matchMedia("(prefers-color-scheme: dark)");
    const handler = (e: MediaQueryListEvent) => {
      setResolvedTheme(e.matches ? "dark" : "light");
      document.documentElement.classList.toggle("dark", e.matches);
    };

    mediaQuery.addEventListener("change", handler);
    return () => mediaQuery.removeEventListener("change", handler);
  }, [theme]);

  return (
    <ThemeContext.Provider value={{ theme, setTheme, resolvedTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error("useTheme must be used within ThemeProvider");
  }
  return context;
}
```

### Theme Toggle Component

```tsx
// theme-toggle.tsx
"use client";

import { useTheme } from "./theme-provider";
import { Sun, Moon, Monitor } from "lucide-react";

export function ThemeToggle() {
  const { theme, setTheme, resolvedTheme } = useTheme();

  return (
    <div className="flex items-center gap-1 p-1 rounded-lg bg-muted">
      <button
        onClick={() => setTheme("light")}
        className={`p-2 rounded-md transition-colors ${
          theme === "light" ? "bg-background shadow-sm" : "hover:bg-background/50"
        }`}
        aria-label="Light mode"
      >
        <Sun className="w-4 h-4" />
      </button>
      <button
        onClick={() => setTheme("dark")}
        className={`p-2 rounded-md transition-colors ${
          theme === "dark" ? "bg-background shadow-sm" : "hover:bg-background/50"
        }`}
        aria-label="Dark mode"
      >
        <Moon className="w-4 h-4" />
      </button>
      <button
        onClick={() => setTheme("system")}
        className={`p-2 rounded-md transition-colors ${
          theme === "system" ? "bg-background shadow-sm" : "hover:bg-background/50"
        }`}
        aria-label="System preference"
      >
        <Monitor className="w-4 h-4" />
      </button>
    </div>
  );
}

// Simple toggle (light/dark only)
export function SimpleThemeToggle() {
  const { resolvedTheme, setTheme } = useTheme();

  return (
    <button
      onClick={() => setTheme(resolvedTheme === "light" ? "dark" : "light")}
      className="p-2 rounded-md hover:bg-muted transition-colors"
      aria-label={`Switch to ${resolvedTheme === "light" ? "dark" : "light"} mode`}
    >
      {resolvedTheme === "light" ? (
        <Moon className="w-5 h-5" />
      ) : (
        <Sun className="w-5 h-5" />
      )}
    </button>
  );
}
```

### Prevent Flash (SSR)

```tsx
// In your root layout or _document
// Add this script to <head> to prevent flash

<script
  dangerouslySetInnerHTML={{
    __html: `
      (function() {
        const theme = localStorage.getItem('theme');
        const systemDark = window.matchMedia('(prefers-color-scheme: dark)').matches;

        if (theme === 'dark' || (theme === 'system' && systemDark) || (!theme && systemDark)) {
          document.documentElement.classList.add('dark');
        }
      })();
    `,
  }}
/>
```

---

## Next.js with next-themes

### Installation & Setup

```bash
npm install next-themes
```

```tsx
// app/providers.tsx
"use client";

import { ThemeProvider } from "next-themes";

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <ThemeProvider
      attribute="class"
      defaultTheme="system"
      enableSystem
      disableTransitionOnChange
    >
      {children}
    </ThemeProvider>
  );
}

// app/layout.tsx
import { Providers } from "./providers";

export default function RootLayout({ children }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

### Using next-themes

```tsx
"use client";

import { useTheme } from "next-themes";

export function ThemeToggle() {
  const { theme, setTheme, resolvedTheme } = useTheme();

  return (
    <button onClick={() => setTheme(resolvedTheme === "dark" ? "light" : "dark")}>
      {resolvedTheme === "dark" ? "🌞" : "🌙"}
    </button>
  );
}
```

---

## Smooth Transitions

### CSS Transitions for Theme Switch

```css
/* Smooth color transitions when switching themes */
:root {
  --transition-theme: background-color 200ms ease, color 200ms ease, border-color 200ms ease;
}

/* Apply to key elements */
body,
.card,
.button,
input,
textarea {
  transition: var(--transition-theme);
}

/* Disable transitions on page load to prevent flash */
.no-transitions * {
  transition: none !important;
}
```

### Animated Toggle with Framer Motion

```tsx
import { motion, AnimatePresence } from "framer-motion";
import { useTheme } from "next-themes";
import { Sun, Moon } from "lucide-react";

export function AnimatedThemeToggle() {
  const { resolvedTheme, setTheme } = useTheme();

  return (
    <button
      onClick={() => setTheme(resolvedTheme === "dark" ? "light" : "dark")}
      className="relative p-2 rounded-full bg-muted overflow-hidden"
    >
      <AnimatePresence mode="wait">
        <motion.div
          key={resolvedTheme}
          initial={{ y: -20, opacity: 0 }}
          animate={{ y: 0, opacity: 1 }}
          exit={{ y: 20, opacity: 0 }}
          transition={{ duration: 0.2 }}
        >
          {resolvedTheme === "dark" ? (
            <Sun className="w-5 h-5" />
          ) : (
            <Moon className="w-5 h-5" />
          )}
        </motion.div>
      </AnimatePresence>
    </button>
  );
}
```

---

## Tailwind CSS Dark Mode

### Configuration

```css
/* Tailwind v4 - app.css */
@import "tailwindcss";

@custom-variant dark (&:where(.dark, .dark *));
```

### Usage

```html
<!-- Background colors -->
<div class="bg-white dark:bg-gray-900">

<!-- Text colors -->
<p class="text-gray-900 dark:text-gray-100">

<!-- Border colors -->
<div class="border-gray-200 dark:border-gray-800">

<!-- Hover states -->
<button class="hover:bg-gray-100 dark:hover:bg-gray-800">

<!-- Complex example -->
<div class="
  bg-white dark:bg-gray-900
  text-gray-900 dark:text-gray-100
  border border-gray-200 dark:border-gray-800
  shadow-lg dark:shadow-gray-900/50
">
  Card content
</div>
```

### Semantic Classes (Recommended)

```css
/* Define semantic colors that auto-switch */
@import "tailwindcss";

@theme {
  --color-background: oklch(100% 0 0);
  --color-foreground: oklch(15% 0 0);
  --color-card: oklch(100% 0 0);
  --color-muted: oklch(96% 0 0);
}

@custom-variant dark (&:where(.dark, .dark *));

.dark {
  --color-background: oklch(12% 0.01 250);
  --color-foreground: oklch(93% 0.01 250);
  --color-card: oklch(15% 0.01 250);
  --color-muted: oklch(20% 0.02 250);
}
```

```html
<!-- Now use semantic colors - auto dark mode! -->
<div class="bg-background text-foreground">
<div class="bg-card">
<p class="text-muted-foreground">
```

---

## Images & Media

### Adaptive Images

```tsx
// Different images for light/dark
function AdaptiveImage() {
  return (
    <>
      <img
        src="/logo-light.svg"
        alt="Logo"
        className="dark:hidden"
      />
      <img
        src="/logo-dark.svg"
        alt="Logo"
        className="hidden dark:block"
      />
    </>
  );
}
```

### CSS Filter Approach

```css
/* Invert images in dark mode */
.dark .invertible-image {
  filter: invert(1) hue-rotate(180deg);
}

/* Reduce brightness of images */
.dark img {
  filter: brightness(0.9);
}
```

### Picture Element

```html
<picture>
  <source srcset="/hero-dark.jpg" media="(prefers-color-scheme: dark)" />
  <img src="/hero-light.jpg" alt="Hero" />
</picture>
```

---

## Accessibility Considerations

### Contrast Requirements

```css
/* Ensure WCAG AA contrast in both modes */

/* Light mode: dark text on light bg */
:root {
  --text-primary: oklch(15% 0 0);    /* Very dark */
  --background: oklch(100% 0 0);      /* White */
  /* Contrast: ~18:1 ✅ */
}

/* Dark mode: light text on dark bg */
.dark {
  --text-primary: oklch(93% 0.01 0);  /* Off-white */
  --background: oklch(12% 0.01 0);    /* Dark gray */
  /* Contrast: ~15:1 ✅ */
}
```

### Focus Indicators

```css
/* Visible in both modes */
:focus-visible {
  outline: 2px solid var(--color-ring);
  outline-offset: 2px;
}
```

### Don't Rely Only on Color

```html
<!-- Include icons/text, not just color changes -->
<span class="flex items-center gap-2">
  <span class="w-2 h-2 rounded-full bg-success" />
  <span>Online</span>
</span>
```

---

## Best Practices Summary

### Do

- Use dark gray (`#121212`) not pure black
- Use off-white not pure white for text
- Desaturate vibrant colors slightly
- Test contrast ratios in both modes
- Provide user toggle AND respect system preference
- Use CSS variables for easy theming
- Add smooth transitions between modes

### Don't

- Invert all colors automatically
- Use pure black backgrounds
- Keep same saturation levels
- Forget about shadows (they don't work on dark)
- Skip testing in actual dark environments
- Flash light mode before dark loads
