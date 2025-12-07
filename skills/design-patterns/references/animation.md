# Animation & Motion

## Animation Timing Guidelines

### Duration by Interaction Type

| Type | Duration | Use Case |
|------|----------|----------|
| Micro | 100-200ms | Hover, focus, button press |
| Small | 200-300ms | Toggles, tabs, dropdowns |
| Medium | 300-500ms | Modals, page transitions |
| Large | 500-1000ms | Complex reveals, hero animations |

### Easing Functions

```css
:root {
  /* Standard easings */
  --ease-linear: linear;
  --ease-in: cubic-bezier(0.4, 0, 1, 1);
  --ease-out: cubic-bezier(0, 0, 0.2, 1);
  --ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);

  /* Spring-like (more natural) */
  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
  --ease-bounce: cubic-bezier(0.68, -0.55, 0.27, 1.55);

  /* Smooth deceleration */
  --ease-smooth: cubic-bezier(0.16, 1, 0.3, 1);
}
```

### When to Use Each Easing

| Easing | Use For |
|--------|---------|
| ease-out | Elements entering (fade in, slide in) |
| ease-in | Elements exiting (fade out, slide out) |
| ease-in-out | State changes, morphing |
| spring | Playful interactions, bouncy UI |
| linear | Progress bars, continuous motion |

---

## CSS Transitions

### Basic Transitions

```css
/* Single property */
.button {
  background-color: var(--color-primary);
  transition: background-color 150ms ease-out;
}

.button:hover {
  background-color: var(--color-primary-hover);
}

/* Multiple properties */
.card {
  transition:
    transform 200ms ease-out,
    box-shadow 200ms ease-out;
}

.card:hover {
  transform: translateY(-4px);
  box-shadow: var(--shadow-lg);
}

/* All properties (use sparingly) */
.element {
  transition: all 200ms ease-out;
}
```

### Transition Tokens

```css
:root {
  /* Duration tokens */
  --duration-fast: 150ms;
  --duration-normal: 200ms;
  --duration-slow: 300ms;

  /* Transition presets */
  --transition-colors: color, background-color, border-color, fill, stroke;
  --transition-transform: transform;
  --transition-opacity: opacity;
  --transition-shadow: box-shadow;
}

.button {
  transition-property: var(--transition-colors);
  transition-duration: var(--duration-fast);
  transition-timing-function: var(--ease-out);
}
```

---

## CSS Animations

### Keyframe Animations

```css
/* Fade in */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

/* Slide up */
@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Scale in */
@keyframes scaleIn {
  from {
    opacity: 0;
    transform: scale(0.95);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

/* Pulse */
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

/* Spin */
@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

/* Shimmer (skeleton loading) */
@keyframes shimmer {
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}
```

### Animation Classes

```css
.animate-fadeIn {
  animation: fadeIn 300ms ease-out forwards;
}

.animate-slideUp {
  animation: slideUp 300ms ease-out forwards;
}

.animate-scaleIn {
  animation: scaleIn 200ms ease-out forwards;
}

.animate-pulse {
  animation: pulse 2s ease-in-out infinite;
}

.animate-spin {
  animation: spin 1s linear infinite;
}

/* Skeleton loading */
.animate-shimmer {
  background: linear-gradient(
    90deg,
    var(--color-muted) 0%,
    var(--color-muted-foreground) 50%,
    var(--color-muted) 100%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s ease-in-out infinite;
}
```

---

## Framer Motion

### Installation

```bash
npm install framer-motion
```

### Basic Animations

```tsx
import { motion } from "framer-motion";

// Fade in on mount
export function FadeIn({ children }) {
  return (
    <motion.div
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      transition={{ duration: 0.3 }}
    >
      {children}
    </motion.div>
  );
}

// Slide up on mount
export function SlideUp({ children }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.3, ease: "easeOut" }}
    >
      {children}
    </motion.div>
  );
}

// Scale on hover
export function ScaleOnHover({ children }) {
  return (
    <motion.div
      whileHover={{ scale: 1.05 }}
      whileTap={{ scale: 0.95 }}
      transition={{ type: "spring", stiffness: 400, damping: 17 }}
    >
      {children}
    </motion.div>
  );
}
```

### Animation Variants

```tsx
const containerVariants = {
  hidden: { opacity: 0 },
  visible: {
    opacity: 1,
    transition: {
      staggerChildren: 0.1,
    },
  },
};

const itemVariants = {
  hidden: { opacity: 0, y: 20 },
  visible: {
    opacity: 1,
    y: 0,
    transition: { duration: 0.3 },
  },
};

function StaggeredList({ items }) {
  return (
    <motion.ul
      variants={containerVariants}
      initial="hidden"
      animate="visible"
    >
      {items.map((item) => (
        <motion.li key={item.id} variants={itemVariants}>
          {item.content}
        </motion.li>
      ))}
    </motion.ul>
  );
}
```

### Exit Animations (AnimatePresence)

```tsx
import { motion, AnimatePresence } from "framer-motion";

function Modal({ isOpen, onClose, children }) {
  return (
    <AnimatePresence>
      {isOpen && (
        <>
          {/* Backdrop */}
          <motion.div
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
            className="fixed inset-0 bg-black/50"
          />

          {/* Modal */}
          <motion.div
            initial={{ opacity: 0, scale: 0.95, y: 20 }}
            animate={{ opacity: 1, scale: 1, y: 0 }}
            exit={{ opacity: 0, scale: 0.95, y: 20 }}
            transition={{ type: "spring", damping: 25, stiffness: 300 }}
            className="fixed inset-x-4 top-1/2 -translate-y-1/2 bg-white rounded-lg p-6"
          >
            {children}
          </motion.div>
        </>
      )}
    </AnimatePresence>
  );
}
```

### Layout Animations

```tsx
// Automatic layout animations
function ExpandableCard({ isExpanded }) {
  return (
    <motion.div
      layout
      transition={{ type: "spring", stiffness: 300, damping: 30 }}
      className={isExpanded ? "h-64" : "h-32"}
    >
      <motion.h2 layout="position">Title</motion.h2>
      {isExpanded && (
        <motion.p
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
        >
          Expanded content
        </motion.p>
      )}
    </motion.div>
  );
}
```

### Spring Physics

```tsx
// Spring configurations
const springConfig = {
  // Snappy (buttons, toggles)
  snappy: { type: "spring", stiffness: 400, damping: 17 },

  // Bouncy (playful UI)
  bouncy: { type: "spring", stiffness: 300, damping: 10 },

  // Smooth (modals, panels)
  smooth: { type: "spring", stiffness: 200, damping: 25 },

  // Slow (page transitions)
  slow: { type: "spring", stiffness: 100, damping: 20 },
};

<motion.div
  animate={{ x: 100 }}
  transition={springConfig.snappy}
/>
```

---

## Micro-Interactions

### Button Press

```tsx
function Button({ children, ...props }) {
  return (
    <motion.button
      whileHover={{ scale: 1.02 }}
      whileTap={{ scale: 0.98 }}
      transition={{ type: "spring", stiffness: 400, damping: 17 }}
      className="px-4 py-2 bg-primary text-white rounded-md"
      {...props}
    >
      {children}
    </motion.button>
  );
}
```

### Toggle Switch

```tsx
function Toggle({ isOn, onToggle }) {
  return (
    <button
      onClick={onToggle}
      className={`w-14 h-8 rounded-full p-1 ${
        isOn ? "bg-primary" : "bg-gray-300"
      }`}
    >
      <motion.div
        className="w-6 h-6 bg-white rounded-full"
        animate={{ x: isOn ? 24 : 0 }}
        transition={{ type: "spring", stiffness: 500, damping: 30 }}
      />
    </button>
  );
}
```

### Checkbox

```tsx
function Checkbox({ checked, onChange }) {
  return (
    <button
      onClick={() => onChange(!checked)}
      className="w-5 h-5 border-2 rounded flex items-center justify-center"
    >
      <motion.svg
        initial={false}
        animate={checked ? "checked" : "unchecked"}
        className="w-3 h-3 text-primary"
        viewBox="0 0 24 24"
      >
        <motion.path
          d="M5 12l5 5L20 7"
          fill="none"
          stroke="currentColor"
          strokeWidth={3}
          variants={{
            unchecked: { pathLength: 0, opacity: 0 },
            checked: { pathLength: 1, opacity: 1 },
          }}
          transition={{ duration: 0.2 }}
        />
      </motion.svg>
    </button>
  );
}
```

### Loading Spinner

```tsx
function Spinner() {
  return (
    <motion.div
      className="w-6 h-6 border-2 border-primary border-t-transparent rounded-full"
      animate={{ rotate: 360 }}
      transition={{ duration: 1, repeat: Infinity, ease: "linear" }}
    />
  );
}
```

### Skeleton Loader

```tsx
function Skeleton({ className }) {
  return (
    <div
      className={`bg-muted animate-pulse rounded ${className}`}
    />
  );
}

function CardSkeleton() {
  return (
    <div className="p-4 space-y-3">
      <Skeleton className="h-4 w-3/4" />
      <Skeleton className="h-4 w-1/2" />
      <Skeleton className="h-20 w-full" />
    </div>
  );
}
```

---

## Scroll Animations

### Scroll-Triggered Animations

```tsx
import { motion, useInView } from "framer-motion";
import { useRef } from "react";

function ScrollReveal({ children }) {
  const ref = useRef(null);
  const isInView = useInView(ref, { once: true, margin: "-100px" });

  return (
    <motion.div
      ref={ref}
      initial={{ opacity: 0, y: 50 }}
      animate={isInView ? { opacity: 1, y: 0 } : { opacity: 0, y: 50 }}
      transition={{ duration: 0.5, ease: "easeOut" }}
    >
      {children}
    </motion.div>
  );
}
```

### Parallax Effect

```tsx
import { motion, useScroll, useTransform } from "framer-motion";

function Parallax({ children }) {
  const { scrollYProgress } = useScroll();
  const y = useTransform(scrollYProgress, [0, 1], [0, -200]);

  return (
    <motion.div style={{ y }}>
      {children}
    </motion.div>
  );
}
```

---

## Accessibility

### Reduced Motion Support

```tsx
import { motion, useReducedMotion } from "framer-motion";

function AccessibleAnimation({ children }) {
  const shouldReduceMotion = useReducedMotion();

  return (
    <motion.div
      initial={{ opacity: 0, y: shouldReduceMotion ? 0 : 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: shouldReduceMotion ? 0 : 0.3 }}
    >
      {children}
    </motion.div>
  );
}
```

### CSS Reduced Motion

```css
/* Respect user preferences */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}

/* Or selectively disable */
@media (prefers-reduced-motion: reduce) {
  .animate-slideUp,
  .animate-fadeIn {
    animation: none;
    opacity: 1;
    transform: none;
  }
}
```

### Focus Animations

```css
/* Accessible focus ring with animation */
.button:focus-visible {
  outline: 2px solid var(--color-ring);
  outline-offset: 2px;
  transition: outline-offset 150ms ease-out;
}

.button:focus-visible:not(:active) {
  outline-offset: 4px;
}
```

---

## Performance Tips

### GPU-Accelerated Properties

```css
/* ✅ Cheap to animate (GPU-accelerated) */
transform: translateX(), translateY(), scale(), rotate();
opacity: 0-1;

/* ❌ Expensive (triggers layout/paint) */
width, height
top, left, right, bottom
margin, padding
font-size
```

### will-change (Use Sparingly)

```css
/* Only for known upcoming animations */
.element-about-to-animate {
  will-change: transform, opacity;
}

/* Remove after animation */
.element-done-animating {
  will-change: auto;
}
```

### Avoid Animation Jank

```tsx
// ❌ Animating layout properties
<motion.div animate={{ width: isOpen ? 300 : 100 }} />

// ✅ Use transform instead
<motion.div animate={{ scaleX: isOpen ? 1 : 0.33 }} />

// ✅ Or use layout animations
<motion.div layout className={isOpen ? "w-[300px]" : "w-[100px]"} />
```
