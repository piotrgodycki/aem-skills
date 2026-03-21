---
title: CSS Best Practices for AEM Edge Delivery Services
impact: HIGH
impactDescription: EDS uses plain vanilla CSS with no build step — applying modern CSS patterns correctly is critical for performance, maintainability, and Lighthouse scores
tags: eds, css, custom-properties, nesting, grid, flexbox, container-queries, performance, scoping, design-tokens
---

## CSS Best Practices for AEM Edge Delivery Services

EDS serves CSS files as-is with zero build step. No SCSS, Less, PostCSS, Tailwind, or any preprocessor is supported. Modern CSS has evolved to cover the vast majority of features that once required preprocessors — custom properties, nesting, container queries, `:has()`, `color-mix()`, `@layer`, and more — all natively in the browser.

---

### 1. Why No Preprocessors in EDS

EDS delivers files directly from a global CDN with no compilation pipeline. This is by design:

- **No build step** means no SCSS, Less, PostCSS, Tailwind, CSS Modules, or styled-components. CSS files in `styles/` and `blocks/` are served exactly as written.
- **Modern CSS covers 90%+ of preprocessor features** — variables (`var()`), nesting, `calc()`, `color-mix()`, `@layer`, and `@container` are all native.
- **Performance benefit** — no compilation overhead, no runtime CSS-in-JS cost, smaller file sizes, instant cache invalidation per file.

EDS targets modern evergreen browsers (Chrome, Firefox, Safari, Edge), so modern CSS features are fully available.

**Incorrect — attempting to use preprocessor syntax:**

```css
/* This will NOT work in EDS — SCSS is not compiled */
$primary-color: #0045ff;
$spacing-md: 1rem;

.hero {
  color: $primary-color;
  padding: $spacing-md;

  &__title {  /* BEM nesting — not needed in EDS */
    font-size: 2rem;
  }
}
```

**Correct — native CSS achieving the same result:**

```css
/* Native CSS custom properties — works in EDS as-is */
:root {
  --color-primary: #0045ff;
  --spacing-md: 1rem;
}

.block.hero {
  color: var(--color-primary);
  padding: var(--spacing-md);

  .title {
    font-size: 2rem;
  }
}
```

---

### 2. CSS Custom Properties (Replacing SCSS Variables)

#### Defining Design Tokens in styles/styles.css

All global design tokens go in `:root` inside `styles/styles.css`. This is the single source of truth for colors, spacing, typography, and layout values:

```css
/* styles/styles.css */
:root {
  /* Colors */
  --color-primary: #0045ff;
  --color-primary-dark: #0033cc;
  --color-secondary: #ff6b00;
  --color-text: #1a1a1a;
  --color-text-muted: #6b7280;
  --color-background: #ffffff;
  --color-surface: #f9fafb;
  --color-border: #e5e7eb;

  /* Typography */
  --font-family-body: 'Adobe Clean', arial, helvetica, sans-serif;
  --font-family-heading: 'Adobe Clean Serif', georgia, serif;
  --font-family-mono: 'Source Code Pro', monospace;
  --font-size-xs: 0.75rem;
  --font-size-sm: 0.875rem;
  --font-size-md: 1rem;
  --font-size-lg: 1.25rem;
  --font-size-xl: 1.5rem;
  --font-size-2xl: 2rem;
  --font-size-3xl: 3rem;
  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-bold: 700;
  --line-height-tight: 1.2;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.75;

  /* Spacing */
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 1.5rem;
  --spacing-xl: 2rem;
  --spacing-2xl: 3rem;
  --spacing-3xl: 4rem;

  /* Layout */
  --content-width: 1200px;
  --content-width-narrow: 800px;
  --grid-gap: 1.5rem;
  --section-spacing: 4rem;

  /* Borders and Radius */
  --border-radius-sm: 4px;
  --border-radius-md: 8px;
  --border-radius-lg: 16px;
  --border-radius-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px rgb(0 0 0 / 5%);
  --shadow-md: 0 4px 6px rgb(0 0 0 / 10%);
  --shadow-lg: 0 10px 25px rgb(0 0 0 / 15%);

  /* Transitions */
  --transition-fast: 150ms ease;
  --transition-normal: 250ms ease;
  --transition-slow: 400ms ease;
}
```

#### Component-Scoped Variables

Blocks can define their own custom properties scoped to that block. This keeps block CSS self-contained and overridable:

**Correct — block-scoped custom properties:**

```css
/* blocks/card/card.css */
.block.card {
  --card-padding: var(--spacing-lg);
  --card-radius: var(--border-radius-md);
  --card-shadow: var(--shadow-sm);
  --card-bg: var(--color-background);

  background: var(--card-bg);
  padding: var(--card-padding);
  border-radius: var(--card-radius);
  box-shadow: var(--card-shadow);

  &:hover {
    --card-shadow: var(--shadow-md);
  }
}
```

This lets sections or themes override card appearance without touching the block CSS:

```css
/* A dark section overrides card variables */
.section.dark .block.card {
  --card-bg: #1a1a2e;
  --card-shadow: 0 4px 12px rgb(0 0 0 / 30%);
}
```

#### Fallback Values

Always provide fallback values for custom properties that might not be defined, especially in reusable blocks:

**Correct — fallback values for resilience:**

```css
.block.teaser {
  color: var(--teaser-text-color, var(--color-text, #1a1a1a));
  padding: var(--teaser-padding, var(--spacing-xl, 2rem));
  background: var(--teaser-bg, transparent);
}
```

**Incorrect — no fallback, breaks if variable is missing:**

```css
.block.teaser {
  color: var(--teaser-text-color);  /* undefined = unset = inherited = unpredictable */
  padding: var(--teaser-padding);
}
```

#### Computed Values with calc()

Custom properties work seamlessly with `calc()` for derived values:

```css
:root {
  --base-size: 1rem;
  --scale-ratio: 1.25;
}

.block.hero {
  /* Derived type scale */
  --title-size: calc(var(--base-size) * var(--scale-ratio) * var(--scale-ratio) * var(--scale-ratio));

  /* Responsive padding based on tokens */
  --block-padding: calc(var(--spacing-xl) + 2vw);

  .title {
    font-size: var(--title-size);
  }

  padding: var(--block-padding);
}
```

#### When to Use Custom Properties vs Hardcoded Values

Use custom properties for:
- Any value that repeats across blocks or components
- Anything an author or theme might want to change
- Colors, spacing, typography, shadows, transitions

Hardcode values when:
- The value is truly one-off and block-internal (e.g., a specific rotation angle in a decorative animation)
- The value is structural and never changes (e.g., `display: grid`)

---

### 3. CSS Nesting (Native — Replacing SCSS Nesting)

Modern browsers support native CSS nesting. EDS targets evergreen browsers, so this is fully available.

#### Basic Nesting Syntax

**Correct — native CSS nesting in a block:**

```css
.block.hero {
  position: relative;
  min-height: 500px;

  .image-wrapper {
    position: absolute;
    inset: 0;

    img {
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
  }

  .content {
    position: relative;
    z-index: 1;
    padding: var(--spacing-2xl);
  }

  .title {
    font-size: var(--font-size-3xl);
    line-height: var(--line-height-tight);
  }

  .description {
    font-size: var(--font-size-lg);
    margin-block-start: var(--spacing-md);
  }
}
```

#### & Selector Usage

The `&` selector references the parent. It is required when appending to the parent selector (pseudo-classes, combined classes) and optional for descendant selectors:

```css
.block.tabs {
  border: 1px solid var(--color-border);

  /* & required: pseudo-class on the block itself */
  &:hover {
    border-color: var(--color-primary);
  }

  /* & required: variant class on the same element */
  &.vertical {
    flex-direction: column;
  }

  /* & optional for descendants — both forms work */
  .tab-item {
    padding: var(--spacing-sm) var(--spacing-md);

    &.active {
      border-bottom: 2px solid var(--color-primary);
      color: var(--color-primary);
    }

    &:focus-visible {
      outline: 2px solid var(--color-primary);
      outline-offset: -2px;
    }
  }
}
```

#### Media Queries Inside Nesting

Media queries can be nested inside selectors, keeping responsive rules co-located with the component they affect:

**Correct — co-located media queries:**

```css
.block.columns {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--grid-gap);

  @media (width >= 768px) {
    grid-template-columns: repeat(2, 1fr);
  }

  @media (width >= 1024px) {
    grid-template-columns: repeat(3, 1fr);
  }

  .column {
    padding: var(--spacing-md);

    @media (width >= 768px) {
      padding: var(--spacing-lg);
    }
  }
}
```

**Incorrect — scattered media queries far from context:**

```css
.block.columns {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--grid-gap);
}

.block.columns .column {
  padding: var(--spacing-md);
}

/* 200 lines later... */
@media (width >= 768px) {
  .block.columns {
    grid-template-columns: repeat(2, 1fr);
  }

  .block.columns .column {
    padding: var(--spacing-lg);
  }
}
```

#### Nesting Depth Limits

Keep nesting to a maximum of 3 levels. Deep nesting creates high specificity, makes CSS harder to read, and signals structural issues:

**Correct — shallow nesting (max 3 levels):**

```css
.block.accordion {
  border: 1px solid var(--color-border);

  .accordion-item {
    border-bottom: 1px solid var(--color-border);

    .accordion-title {
      font-weight: var(--font-weight-bold);
    }
  }
}
```

**Incorrect — too deeply nested:**

```css
.block.accordion {
  .accordion-list {
    .accordion-item {
      .accordion-header {
        .accordion-title {
          .accordion-icon {   /* 6 levels deep — unreadable, overly specific */
            transform: rotate(90deg);
          }
        }
      }
    }
  }
}
```

---

### 4. Modern CSS Layout

#### CSS Grid for Complex Layouts

Use CSS Grid for two-dimensional layouts — rows and columns together:

```css
.block.feature-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: var(--grid-gap);
  padding: var(--spacing-xl);

  .feature-item {
    display: grid;
    grid-template-rows: auto 1fr auto;
    gap: var(--spacing-sm);
  }
}
```

#### Flexbox for 1D Alignment

Use Flexbox when you only need alignment in one direction:

```css
.block.breadcrumb {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: var(--spacing-sm);
  padding: var(--spacing-sm) 0;

  .breadcrumb-item {
    display: flex;
    align-items: center;
    gap: var(--spacing-sm);
  }
}
```

#### Container Queries for Component-Level Responsive Design

Container queries let a block respond to its own container size rather than the viewport. This is essential for blocks that appear in different layout contexts (full-width vs sidebar):

```css
.block.card {
  container-type: inline-size;
  container-name: card;

  .card-content {
    display: flex;
    flex-direction: column;
    gap: var(--spacing-md);
  }

  /* When the card container is wide enough, switch to horizontal layout */
  @container card (inline-size > 500px) {
    .card-content {
      flex-direction: row;
      align-items: center;
    }

    .card-image {
      flex: 0 0 40%;
    }

    .card-text {
      flex: 1;
    }
  }
}
```

#### Fluid Typography and Spacing with clamp()

Use `clamp()` for fluid values that scale between a minimum and maximum:

```css
:root {
  /* Fluid type scale — no media query breakpoints needed */
  --font-size-fluid-sm: clamp(0.875rem, 0.8rem + 0.25vw, 1rem);
  --font-size-fluid-md: clamp(1rem, 0.9rem + 0.5vw, 1.25rem);
  --font-size-fluid-lg: clamp(1.25rem, 1rem + 1vw, 2rem);
  --font-size-fluid-xl: clamp(2rem, 1.5rem + 2vw, 3.5rem);

  /* Fluid spacing */
  --spacing-fluid-section: clamp(2rem, 1.5rem + 3vw, 6rem);
}

.block.hero .title {
  font-size: var(--font-size-fluid-xl);
  line-height: var(--line-height-tight);
}
```

**Incorrect — fixed breakpoints for every font size:**

```css
.block.hero .title {
  font-size: 1.5rem;
}

@media (width >= 768px) {
  .block.hero .title { font-size: 2rem; }
}

@media (width >= 1024px) {
  .block.hero .title { font-size: 2.5rem; }
}

@media (width >= 1280px) {
  .block.hero .title { font-size: 3.5rem; }
}
```

#### Aspect-Ratio for Media Containers

```css
.block.video .video-wrapper {
  aspect-ratio: 16 / 9;
  width: 100%;
  overflow: hidden;
  border-radius: var(--border-radius-md);

  iframe,
  video {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
}

.block.gallery .thumbnail {
  aspect-ratio: 1;
  object-fit: cover;
}
```

#### Gap Property (Replacing Margin Hacks)

**Correct — gap for spacing between items:**

```css
.block.tags {
  display: flex;
  flex-wrap: wrap;
  gap: var(--spacing-sm);
}
```

**Incorrect — margin hacks with negative margin wrappers:**

```css
.block.tags {
  display: flex;
  flex-wrap: wrap;
  margin: -4px;
}

.block.tags .tag {
  margin: 4px;  /* Fragile, causes overflow issues */
}
```

#### Logical Properties

Use logical properties (`inline`/`block`) instead of physical directions (`left`/`right`/`top`/`bottom`). This supports RTL languages and writing-mode changes:

**Correct — logical properties:**

```css
.block.sidebar {
  padding-inline: var(--spacing-lg);
  padding-block: var(--spacing-xl);
  margin-block-end: var(--spacing-2xl);
  border-inline-start: 4px solid var(--color-primary);
}
```

**Incorrect — physical properties only:**

```css
.block.sidebar {
  padding-left: var(--spacing-lg);
  padding-right: var(--spacing-lg);
  padding-top: var(--spacing-xl);
  padding-bottom: var(--spacing-xl);
  margin-bottom: var(--spacing-2xl);
  border-left: 4px solid var(--color-primary);
}
```

---

### 5. CSS Scoping in EDS

EDS uses a class-based scoping model. Every block gets a wrapper `<div>` with two classes: `block` and the block name. This is the foundation for scoping.

#### Block Scoping

Always scope block styles with `.block.<name>`:

**Correct — scoped to the specific block:**

```css
.block.teaser {
  position: relative;
  overflow: hidden;

  .title {
    font-size: var(--font-size-2xl);
  }

  .description {
    color: var(--color-text-muted);
  }

  .button {
    margin-block-start: var(--spacing-md);
  }
}
```

**Incorrect — unscoped, leaks globally:**

```css
.title {
  font-size: var(--font-size-2xl);  /* Affects EVERY .title on the page */
}

.description {
  color: var(--color-text-muted);
}
```

**Incorrect — using only the block name without .block:**

```css
.teaser {
  /* Less specific than .block.teaser — could be overridden unexpectedly */
  position: relative;
}
```

#### Section Scoping

Section metadata adds classes to the `.section` wrapper. Use `.section.<class>` for section-level styles:

```css
/* styles/styles.css or a block CSS */
.section.highlight {
  background: var(--color-surface);
  padding-block: var(--section-spacing);
}

.section.dark {
  --color-text: #ffffff;
  --color-text-muted: #d1d5db;
  --color-background: #1a1a2e;
  --color-border: #374151;

  background: var(--color-background);
  color: var(--color-text);
}

.section.narrow {
  max-width: var(--content-width-narrow);
  margin-inline: auto;
}
```

#### Default Content Styling

Content outside of blocks (bare paragraphs, headings, lists in sections) is wrapped in `.default-content-wrapper`:

```css
/* styles/styles.css */
.default-content-wrapper {
  max-width: var(--content-width);
  margin-inline: auto;
  padding-inline: var(--spacing-lg);

  h1, h2, h3, h4, h5, h6 {
    font-family: var(--font-family-heading);
    line-height: var(--line-height-tight);
    margin-block: var(--spacing-lg) var(--spacing-sm);
  }

  p {
    margin-block: 0 var(--spacing-md);
    line-height: var(--line-height-normal);
  }

  ul, ol {
    padding-inline-start: var(--spacing-xl);
    margin-block: 0 var(--spacing-md);
  }

  a {
    color: var(--color-primary);
    text-decoration: underline;
    text-underline-offset: 2px;

    &:hover {
      color: var(--color-primary-dark);
    }
  }
}
```

#### Specificity Management

EDS does not use BEM. Instead, rely on the natural DOM structure and block scoping:

**Correct — EDS-style scoping:**

```css
.block.card {
  .image { ... }
  .title { ... }
  .description { ... }
  .cta { ... }
}
```

**Incorrect — BEM naming (unnecessary in EDS):**

```css
.card__image { ... }
.card__title { ... }
.card__title--large { ... }
.card__description { ... }
```

The `.block.<name>` scoping pattern provides enough specificity to avoid conflicts. If two blocks both have a `.title` child, their styles do not collide because each is scoped:

```css
/* These do NOT conflict */
.block.hero .title { font-size: var(--font-size-3xl); }
.block.card .title { font-size: var(--font-size-lg); }
```

---

### 6. CSS for Performance

#### content-visibility for Offscreen Content

Use `content-visibility: auto` on blocks below the fold to skip rendering until they scroll into view:

```css
.block.faq,
.block.footer,
.block.related-articles {
  content-visibility: auto;
  contain-intrinsic-size: auto 500px; /* Estimated height to prevent layout shift */
}
```

#### will-change (Use Sparingly)

Only apply `will-change` to elements that actually animate, and prefer adding it via JS just before animation:

**Correct — scoped and purposeful:**

```css
.block.carousel .slide {
  transition: transform var(--transition-normal);
}

.block.carousel .slide.animating {
  will-change: transform;
}
```

**Incorrect — blanket will-change on everything:**

```css
.block.carousel * {
  will-change: transform, opacity;  /* Wastes GPU memory for every element */
}
```

#### Prefers-Reduced-Motion

Always respect the user's motion preferences:

```css
.block.hero .image {
  transition: transform var(--transition-slow);

  &:hover {
    transform: scale(1.05);
  }
}

@media (prefers-reduced-motion: reduce) {
  .block.hero .image {
    transition: none;

    &:hover {
      transform: none;
    }
  }
}
```

Or use a modern approach — only enable motion for users who want it:

```css
.block.hero .image {
  /* No motion by default */
}

@media (prefers-reduced-motion: no-preference) {
  .block.hero .image {
    transition: transform var(--transition-slow);

    &:hover {
      transform: scale(1.05);
    }
  }
}
```

#### Prefers-Color-Scheme for Dark Mode

```css
@media (prefers-color-scheme: dark) {
  :root {
    --color-text: #f3f4f6;
    --color-text-muted: #9ca3af;
    --color-background: #111827;
    --color-surface: #1f2937;
    --color-border: #374151;
    --color-primary: #60a5fa;
    --shadow-sm: 0 1px 2px rgb(0 0 0 / 20%);
    --shadow-md: 0 4px 6px rgb(0 0 0 / 30%);
  }
}
```

#### Font-Display Strategy

```css
/* styles/fonts.css */
@font-face {
  font-family: 'Adobe Clean';
  src: url('/fonts/adobe-clean-regular.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;   /* Show fallback immediately, swap when loaded */
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+02BB-02BC, U+2000-206F;
}

@font-face {
  font-family: 'Adobe Clean';
  src: url('/fonts/adobe-clean-bold.woff2') format('woff2');
  font-weight: 700;
  font-style: normal;
  font-display: swap;
}
```

#### Avoiding Expensive Selectors

**Correct — specific, shallow selectors:**

```css
.block.card .title { ... }
.block.card .image img { ... }
```

**Incorrect — expensive universal and deep selectors:**

```css
* { box-sizing: border-box; }         /* Apply once in styles.css on html, not everywhere */
.block.card div div div p { ... }      /* Over-qualified, fragile */
[class*="icon-"] { ... }               /* Attribute selectors are slower */
```

---

### 7. Modern CSS Features Replacing JS

#### :has() Selector (Parent Selector)

Select parents based on their children — no JavaScript needed:

```css
/* Card with an image gets different layout */
.block.card:has(picture) {
  display: grid;
  grid-template-columns: 1fr 1fr;
}

/* Card without an image is full-width text */
.block.card:not(:has(picture)) {
  max-width: 65ch;
}

/* Section with a hero block removes its default padding */
.section:has(.block.hero) {
  padding: 0;
}

/* Form field with an invalid input shows error styling */
.field:has(input:invalid) {
  .label {
    color: var(--color-error);
  }
}
```

#### :is() and :where() for Grouping

`:is()` matches any selector in its list. `:where()` does the same but with zero specificity:

```css
/* :is() — groups selectors with the highest specificity of the group */
.block.text-content :is(h1, h2, h3, h4) {
  font-family: var(--font-family-heading);
  line-height: var(--line-height-tight);
}

/* :where() — zero specificity, easy to override */
:where(.block.text-content) :where(p, li, dd) {
  line-height: var(--line-height-normal);
  max-width: 65ch;
}
```

#### @layer for Style Ordering

Use `@layer` to control cascade order without fighting specificity. Define layers once, and later rules within lower layers cannot override higher layers regardless of specificity:

```css
/* styles/styles.css — define layer order once at the top */
@layer reset, tokens, base, blocks, sections, utilities;

@layer reset {
  *,
  *::before,
  *::after {
    box-sizing: border-box;
    margin: 0;
  }
}

@layer tokens {
  :root {
    --color-primary: #0045ff;
    /* ... all design tokens ... */
  }
}

@layer base {
  body {
    font-family: var(--font-family-body);
    color: var(--color-text);
    background: var(--color-background);
  }
}

@layer utilities {
  .visually-hidden {
    clip: rect(0 0 0 0);
    clip-path: inset(50%);
    height: 1px;
    overflow: hidden;
    position: absolute;
    white-space: nowrap;
    width: 1px;
  }
}
```

#### color-mix() for Dynamic Colors

Generate color variations without preprocessors:

```css
:root {
  --color-primary: #0045ff;
  --color-primary-light: color-mix(in oklch, var(--color-primary), white 30%);
  --color-primary-dark: color-mix(in oklch, var(--color-primary), black 20%);
  --color-primary-translucent: color-mix(in srgb, var(--color-primary) 15%, transparent);
}

.block.hero {
  background: var(--color-primary);

  &:hover {
    background: var(--color-primary-dark);
  }

  .badge {
    background: var(--color-primary-translucent);
  }
}
```

#### oklch/oklab Color Spaces

`oklch` provides perceptually uniform colors — adjusting lightness or chroma does not shift hue:

```css
:root {
  /* Define a hue, then derive a full palette */
  --hue-brand: 264;

  --color-brand-50: oklch(97% 0.02 var(--hue-brand));
  --color-brand-100: oklch(93% 0.05 var(--hue-brand));
  --color-brand-500: oklch(55% 0.25 var(--hue-brand));
  --color-brand-700: oklch(40% 0.2 var(--hue-brand));
  --color-brand-900: oklch(25% 0.12 var(--hue-brand));
}
```

#### Scroll-Driven Animations

Animate elements based on scroll position without JavaScript:

```css
.block.hero .image {
  animation: parallax linear;
  animation-timeline: scroll();
  animation-range: 0vh 80vh;
}

@keyframes parallax {
  from { transform: translateY(0); }
  to { transform: translateY(-60px); }
}

/* Fade in as block scrolls into view */
.block.stats {
  animation: fade-in linear;
  animation-timeline: view();
  animation-range: entry 0% entry 100%;
}

@keyframes fade-in {
  from { opacity: 0; transform: translateY(30px); }
  to { opacity: 1; transform: translateY(0); }
}
```

#### @scope for Scoping

`@scope` limits styles to a subtree of the DOM, with optional lower bounds:

```css
/* Styles apply inside .block.article but not inside .block.sidebar nested within */
@scope (.block.article) to (.block.sidebar) {
  p {
    max-width: 65ch;
    line-height: var(--line-height-relaxed);
  }

  a {
    color: var(--color-primary);
    text-decoration: underline;
  }
}
```

#### Popover API Styling

```css
.block.dropdown [popover] {
  border: 1px solid var(--color-border);
  border-radius: var(--border-radius-md);
  padding: var(--spacing-md);
  box-shadow: var(--shadow-lg);

  &::backdrop {
    background: rgb(0 0 0 / 20%);
  }
}
```

---

### 8. EDS-Specific CSS Patterns

#### File Organization

| File | Purpose |
|------|---------|
| `styles/styles.css` | Global design tokens, base element styles, section styles, utility classes |
| `styles/fonts.css` | `@font-face` declarations — loaded asynchronously |
| `blocks/<name>/<name>.css` | Block-specific styles — auto-loaded when block appears on page |

#### Auto-Loading: CSS Loaded Only When Block Is On Page

EDS automatically loads `blocks/<name>/<name>.css` only when the block is present on the page. You do not need to manually import block CSS. This means:

- Block CSS files are **lazy-loaded** — they do not penalize pages that don't use the block.
- Global styles that apply everywhere go in `styles/styles.css`.
- Never `@import` block CSS files from `styles/styles.css`.

#### Variant Styles via Block Options

Block options (classes from the block table) are added to the block wrapper. Style variants accordingly:

```css
/* blocks/hero/hero.css */
.block.hero {
  min-height: 500px;
  color: var(--color-text);
  background: var(--color-background);

  /* Variant: dark */
  &.dark {
    --color-text: #ffffff;
    --color-background: #1a1a2e;
    color: var(--color-text);
    background: var(--color-background);
  }

  /* Variant: centered */
  &.centered {
    text-align: center;

    .content {
      align-items: center;
      max-width: 800px;
      margin-inline: auto;
    }
  }

  /* Variant: short */
  &.short {
    min-height: 300px;
  }

  /* Multiple variants can combine: .block.hero.dark.centered */
}
```

#### Section Metadata Styles

Section metadata in the document adds classes to the wrapping `.section` element:

```css
/* styles/styles.css */

/* Section style: background variants */
.section.light-gray {
  background: var(--color-surface);
}

.section.dark {
  background: var(--color-background);
  color: var(--color-text);

  /* Override tokens for all blocks inside */
  --color-text: #f3f4f6;
  --color-text-muted: #9ca3af;
  --color-background: #111827;
  --color-border: #374151;
}

/* Section style: width */
.section.narrow .section-wrapper {
  max-width: var(--content-width-narrow);
}
```

#### Icon Styling

EDS loads icons as inline SVGs using `.icon` and `.icon-<name>` classes:

```css
/* styles/styles.css */
.icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;

  svg {
    width: 1em;
    height: 1em;
    fill: currentcolor;
    vertical-align: middle;
  }
}

/* Size variants */
.icon.icon-lg svg {
  width: 1.5em;
  height: 1.5em;
}

/* Specific icon color override */
.icon.icon-checkmark svg {
  fill: var(--color-success, #16a34a);
}
```

#### Button Styling

EDS auto-wraps links in certain contexts with `.button-container` and `.button` classes:

```css
/* styles/styles.css */
.button {
  display: inline-flex;
  align-items: center;
  gap: var(--spacing-sm);
  padding: var(--spacing-sm) var(--spacing-lg);
  font-size: var(--font-size-md);
  font-weight: var(--font-weight-medium);
  line-height: var(--line-height-normal);
  text-decoration: none;
  border: 2px solid transparent;
  border-radius: var(--border-radius-sm);
  cursor: pointer;
  transition: background var(--transition-fast), color var(--transition-fast), border-color var(--transition-fast);

  /* Primary (default) */
  background: var(--color-primary);
  color: #ffffff;

  &:hover {
    background: var(--color-primary-dark);
  }

  &:focus-visible {
    outline: 2px solid var(--color-primary);
    outline-offset: 2px;
  }

  /* Secondary variant */
  &.secondary {
    background: transparent;
    color: var(--color-primary);
    border-color: var(--color-primary);

    &:hover {
      background: var(--color-primary);
      color: #ffffff;
    }
  }
}
```

---

### 9. Anti-Patterns

#### Avoid !important

**Incorrect — using !important to fight specificity:**

```css
.block.card .title {
  color: red !important;   /* Almost always a sign of a scoping problem */
}
```

**Correct — fix the scoping instead:**

```css
/* If a section override is too specific, match that specificity */
.section.dark .block.card .title {
  color: var(--color-text);
}
```

#### No ID Selectors for Styling

**Incorrect — IDs create unreasonably high specificity:**

```css
#main-hero { ... }
#navigation .link { ... }
```

**Correct — class-based selectors:**

```css
.block.hero { ... }
.block.navigation .link { ... }
```

#### No Inline Styles from JS

**Incorrect — setting styles directly:**

```javascript
export default function decorate(block) {
  block.style.backgroundColor = 'red';
  block.querySelector('.title').style.fontSize = '2rem';
}
```

**Correct — toggle classes and let CSS handle presentation:**

```javascript
export default function decorate(block) {
  block.classList.add('active');
  block.querySelector('.title')?.classList.add('large');
}
```

```css
.block.hero.active {
  background: var(--color-primary);
}

.block.hero .title.large {
  font-size: var(--font-size-2xl);
}
```

#### No Absolute Positioning for Layout

**Incorrect — using absolute positioning to build a grid:**

```css
.block.features .item:nth-child(1) { position: absolute; top: 0; left: 0; width: 50%; }
.block.features .item:nth-child(2) { position: absolute; top: 0; left: 50%; width: 50%; }
.block.features .item:nth-child(3) { position: absolute; top: 300px; left: 0; width: 100%; }
```

**Correct — CSS Grid:**

```css
.block.features {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: var(--grid-gap);

  .item:last-child {
    grid-column: 1 / -1;
  }
}
```

#### Use Relative Units, Not Fixed Pixels

**Incorrect — hardcoded pixel values everywhere:**

```css
.block.sidebar {
  width: 350px;
  padding: 24px;
  font-size: 14px;
  margin-bottom: 32px;
}
```

**Correct — relative units with tokens:**

```css
.block.sidebar {
  width: min(350px, 100%);
  padding: var(--spacing-lg);
  font-size: var(--font-size-sm);
  margin-block-end: var(--spacing-2xl);
}
```

#### No Vendor Prefixes

EDS targets modern evergreen browsers. Vendor prefixes are unnecessary:

**Incorrect — outdated vendor prefixes:**

```css
.block.card {
  -webkit-border-radius: 8px;
  -moz-border-radius: 8px;
  border-radius: 8px;
  display: -webkit-flex;
  display: -ms-flexbox;
  display: flex;
  -webkit-transition: transform 0.3s;
  transition: transform 0.3s;
}
```

**Correct — standards only:**

```css
.block.card {
  border-radius: var(--border-radius-md);
  display: flex;
  transition: transform var(--transition-normal);
}
```

---

### Quick Reference: Preprocessor Feature to Native CSS

| Preprocessor Feature | Native CSS Equivalent |
|---|---|
| `$variable` | `var(--variable)` |
| SCSS nesting | Native CSS nesting |
| `@mixin` / `@include` | Custom properties + `@layer` |
| `lighten()` / `darken()` | `color-mix()` |
| `@extend` | `:is()` / `:where()` grouping |
| `@import` partials | `@layer` + auto-loaded block CSS |
| `@each` / `@for` loops | Repeated rules or JS-generated custom properties |
| `math.div()` / `percentage()` | `calc()` / native `%` |
| Vendor prefix autoprefixer | Not needed (modern browsers) |
| Conditional logic | `:has()`, `:is()`, `@supports` |
