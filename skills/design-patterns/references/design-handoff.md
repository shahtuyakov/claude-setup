# Design Handoff

## Figma Dev Mode

Dev Mode is Figma's dedicated workspace for developers, providing detailed specifications, measurements, and code snippets for designs.

### Accessing Dev Mode

```
Toggle: Press Shift + D or click the </> icon in toolbar
Available: On all Figma plans (limited features on free)
```

### Key Features

| Feature | Description |
|---------|-------------|
| Inspect Panel | View spacing, colors, typography specs |
| Code Snippets | Copy CSS, iOS (SwiftUI), Android (Compose) |
| Variables | See design tokens/variables used |
| Ready for Dev | Status indicator for dev-ready designs |
| Annotations | Designer notes and context |
| Compare Changes | Diff view between versions |

---

## Extracting Design Specifications

### Colors

```
In Dev Mode → Select element → Inspect panel → Fill/Stroke

Example output:
Fill: #3B82F6 (or oklch(55% 0.25 250) if using modern colors)
Opacity: 100%

Convert to CSS:
background-color: #3B82F6;
/* or */
background-color: oklch(55% 0.25 250);
```

### Typography

```
In Dev Mode → Select text → Inspect panel

Example output:
Font: Inter
Weight: 600 (Semi Bold)
Size: 24px
Line height: 32px (133%)
Letter spacing: -0.02em

Convert to CSS:
font-family: 'Inter', sans-serif;
font-weight: 600;
font-size: 24px;
line-height: 32px;
letter-spacing: -0.02em;

/* Or with Tailwind: */
className="font-semibold text-2xl leading-8 tracking-tight"
```

### Spacing & Layout

```
In Dev Mode → Select element → Inspect panel

Example output:
Width: 320px
Height: Auto
Padding: 24px 16px
Gap: 12px
Border radius: 8px

Convert to CSS:
width: 320px;
padding: 24px 16px;
gap: 12px;
border-radius: 8px;

/* Tailwind: */
className="w-80 px-4 py-6 gap-3 rounded-lg"
```

### Shadows

```
In Dev Mode → Select element → Effects

Example output:
Drop shadow: 0px 4px 6px rgba(0, 0, 0, 0.1)

Convert to CSS:
box-shadow: 0px 4px 6px rgba(0, 0, 0, 0.1);

/* Tailwind: */
className="shadow-md"
```

---

## Figma Variables to CSS

### Reading Variables in Dev Mode

```
1. Select element in Dev Mode
2. Look for variable pill (e.g., "colors/primary/500")
3. Click to see resolved value

Example:
Variable: colors/primary/500
Resolved: #3B82F6
```

### Mapping to CSS Variables

```css
/* Figma Variables → CSS Custom Properties */

/* Figma: colors/primary/50 - 950 */
:root {
  --color-primary-50: #eff6ff;
  --color-primary-100: #dbeafe;
  --color-primary-200: #bfdbfe;
  --color-primary-500: #3b82f6;
  --color-primary-900: #1e3a8a;
}

/* Figma: spacing/1 - 12 */
:root {
  --spacing-1: 4px;
  --spacing-2: 8px;
  --spacing-3: 12px;
  --spacing-4: 16px;
  --spacing-6: 24px;
  --spacing-8: 32px;
}

/* Figma: radius/sm, md, lg */
:root {
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
}
```

---

## Code Connect

Code Connect links your actual codebase components to Figma designs, providing real code snippets instead of generated CSS.

### Setup

```bash
# Install Figma CLI
npm install -g @figma/code-connect

# Initialize in your project
figma connect init
```

### Configuration

```typescript
// figma.config.ts
import { figmaConnect } from "@figma/code-connect";

export default {
  // Your Figma file URL
  figmaFileUrl: "https://www.figma.com/file/xxx/Design-System",

  // Component mappings
  components: {
    Button: {
      figmaNodeId: "123:456",
      component: "@/components/ui/button",
    },
  },
};
```

### Component Mapping

```tsx
// button.figma.tsx
import figma from "@figma/code-connect";
import { Button } from "./button";

figma.connect(Button, "https://figma.com/file/xxx?node-id=123:456", {
  props: {
    variant: figma.enum("Variant", {
      Primary: "default",
      Secondary: "secondary",
      Outline: "outline",
    }),
    size: figma.enum("Size", {
      Small: "sm",
      Medium: "default",
      Large: "lg",
    }),
    label: figma.string("Label"),
    disabled: figma.boolean("Disabled"),
  },
  example: (props) => (
    <Button variant={props.variant} size={props.size} disabled={props.disabled}>
      {props.label}
    </Button>
  ),
});
```

### Result in Figma

```
When developer selects the Button in Figma Dev Mode:

Instead of:
  background-color: #3B82F6;
  padding: 8px 16px;
  border-radius: 6px;
  ...

They see:
  <Button variant="default" size="default">
    Click me
  </Button>
```

---

## Tokens Studio Integration

### Export from Figma

```json
// tokens.json (W3C DTCG format)
{
  "colors": {
    "primary": {
      "50": {
        "$type": "color",
        "$value": "#eff6ff"
      },
      "500": {
        "$type": "color",
        "$value": "#3b82f6"
      }
    }
  },
  "spacing": {
    "4": {
      "$type": "dimension",
      "$value": "16px"
    }
  }
}
```

### Transform with Style Dictionary

```javascript
// style-dictionary.config.js
export default {
  source: ["tokens/**/*.json"],
  platforms: {
    css: {
      transformGroup: "css",
      buildPath: "src/styles/",
      files: [{
        destination: "variables.css",
        format: "css/variables",
      }],
    },
    tailwind: {
      transformGroup: "js",
      buildPath: "tailwind/",
      files: [{
        destination: "tokens.js",
        format: "javascript/es6",
      }],
    },
  },
};
```

### Output

```css
/* variables.css */
:root {
  --colors-primary-50: #eff6ff;
  --colors-primary-500: #3b82f6;
  --spacing-4: 16px;
}
```

---

## Handoff Checklist

### Designer Checklist

Before marking "Ready for Dev":

- [ ] **Components** using design system components
- [ ] **Variables** applied (not raw values)
- [ ] **Auto Layout** for all flexible containers
- [ ] **Constraints** set for responsive behavior
- [ ] **States** defined (hover, active, disabled, focus)
- [ ] **Annotations** added for complex interactions
- [ ] **Variants** organized in component sets
- [ ] **Naming** consistent and meaningful

### Developer Checklist

When implementing:

- [ ] **Match tokens** to CSS variables
- [ ] **Verify responsive** behavior at breakpoints
- [ ] **Check states** (hover, focus, disabled)
- [ ] **Test accessibility** (contrast, focus rings)
- [ ] **Confirm animations** timing/easing
- [ ] **Review edge cases** (empty, loading, error)

---

## Reading Figma Auto Layout

### Auto Layout → CSS Flexbox

```
Figma Auto Layout:
  Direction: Horizontal
  Gap: 12px
  Padding: 16px
  Alignment: Center

CSS:
  display: flex;
  flex-direction: row;
  gap: 12px;
  padding: 16px;
  align-items: center;
  justify-content: flex-start;

Tailwind:
  className="flex flex-row gap-3 p-4 items-center"
```

### Alignment Mapping

| Figma | CSS (align-items) | CSS (justify-content) |
|-------|-------------------|----------------------|
| Top-left | flex-start | flex-start |
| Top-center | flex-start | center |
| Top-right | flex-start | flex-end |
| Center-left | center | flex-start |
| Center | center | center |
| Center-right | center | flex-end |
| Bottom-left | flex-end | flex-start |
| Space between | - | space-between |

### Fill Container vs Hug Contents

```
Figma "Fill container" (horizontal):
  width: 100%;
  /* or */
  flex: 1;

Figma "Hug contents":
  width: auto;
  /* or */
  flex: none;

Figma "Fixed":
  width: 200px;
```

---

## Responsive Design Specs

### Reading Constraints

```
Figma Constraints:
  Left & Right: Scale

CSS:
  position: absolute;
  left: 16px;
  right: 16px;
  /* or */
  width: calc(100% - 32px);
  margin: 0 auto;
```

### Breakpoint Annotations

Look for designer annotations indicating:
- Mobile: 375px
- Tablet: 768px
- Desktop: 1280px

```css
/* Implement with media queries */
.container {
  padding: 16px;
}

@media (min-width: 768px) {
  .container {
    padding: 24px;
  }
}

@media (min-width: 1280px) {
  .container {
    padding: 32px;
    max-width: 1200px;
  }
}
```

---

## Exporting Assets

### Images

```
Right-click → Export
Formats: PNG, JPG, SVG, WebP
Scales: 1x, 2x, 3x (for retina)

Recommended:
- Icons: SVG
- Photos: WebP with JPG fallback
- UI elements: SVG or PNG@2x
```

### Icons as SVG

```
1. Select icon frame
2. Right-click → Copy as SVG
3. Paste into code

Or export and use as:
<img src="/icons/icon.svg" alt="" />

Or inline:
<svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
  <path d="M12 5v14M5 12h14" />
</svg>
```

---

## Communication Patterns

### Asking Clarifying Questions

```markdown
## Design Questions for [Feature Name]

### Interactions
- What happens when user hovers on card?
- Is there a loading state for this button?
- What's the transition duration for modal?

### Edge Cases
- How does this look with very long text?
- What's the empty state?
- Error state design?

### Responsive
- How does this adapt on mobile?
- Is this component visible on tablet?

### Accessibility
- What's the focus order?
- Any ARIA labels needed?
```

### Documenting Deviations

```markdown
## Implementation Notes

### Deviation from Design
**Component**: Card hover effect
**Design**: Scale to 1.05
**Implementation**: Scale to 1.02
**Reason**: 1.05 felt too aggressive and caused layout shift

### Technical Constraints
**Component**: Blur background
**Design**: backdrop-filter: blur(20px)
**Implementation**: backdrop-filter: blur(12px)
**Reason**: Performance issues on mobile Safari
```

---

## IDE Integration

### VS Code Extension

```
1. Install "Figma for VS Code" extension
2. Link Figma account
3. Select component in Figma
4. View specs in VS Code sidebar
```

### Figma MCP (AI Integration)

```
Figma MCP server enables AI coding tools to:
- Read design context directly
- Generate code matching design specs
- Access component properties and variants

Works with: VS Code, Cursor, Windsurf, Claude
```

---

## Best Practices

### For Designers

1. **Use variables** - Always use design tokens, not raw values
2. **Name layers** - Meaningful names help developers
3. **Document states** - Show all interactive states
4. **Add annotations** - Explain complex interactions
5. **Set Ready status** - Mark when dev-ready

### For Developers

1. **Check variables first** - Match to existing tokens
2. **Ask before improvising** - Clarify unclear specs
3. **Document deviations** - Record any changes made
4. **Test on device** - Verify responsive behavior
5. **Share implementation** - Screenshot/video for review
