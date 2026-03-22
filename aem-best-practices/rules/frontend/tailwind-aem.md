---
title: Tailwind CSS for AEM
impact: HIGH
impactDescription: Tailwind in AEM requires careful integration with the ClientLib pipeline, HTL templating, and Core Component BEM classes to avoid style conflicts and bloated bundles
tags: tailwind, css, aem, utility-first, clientlibs, htl, purge, jit, webpack, postcss
---

## Tailwind CSS for AEM

Tailwind CSS brings utility-first styling to AEM projects. Integration requires PostCSS in the `ui.frontend` Webpack pipeline, proper content scanning for purging, and a clear strategy for coexisting with Core Component BEM classes.

---

### 1. Installation & Webpack Setup

```bash
cd ui.frontend
npm install -D tailwindcss @tailwindcss/postcss postcss postcss-loader
npx tailwindcss init
```

**PostCSS config:**

```javascript
// ui.frontend/postcss.config.js
module.exports = {
  plugins: [
    '@tailwindcss/postcss',
    'autoprefixer',
    ...(process.env.NODE_ENV === 'production' ? [['cssnano', { preset: 'default' }]] : []),
  ],
};
```

**Webpack integration:**

```javascript
// ui.frontend/webpack.common.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          MiniCssExtractPlugin.loader,
          { loader: 'css-loader', options: { sourceMap: true } },
          { loader: 'postcss-loader', options: { sourceMap: true } },
        ],
      },
      // SCSS still works alongside Tailwind
      {
        test: /\.scss$/,
        use: [
          MiniCssExtractPlugin.loader,
          { loader: 'css-loader', options: { sourceMap: true } },
          { loader: 'postcss-loader', options: { sourceMap: true } },
          { loader: 'sass-loader', options: { sourceMap: true } },
        ],
      },
    ],
  },
};
```

---

### 2. Tailwind Config for AEM

```javascript
// ui.frontend/tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    // Scan HTL templates for Tailwind classes
    '../ui.apps/src/main/content/jcr_root/apps/mysite/**/*.html',
    // Scan JS/TSX files
    './src/**/*.{js,ts,jsx,tsx}',
    // Scan HTL data-sly-attribute and dialog XML for dynamic classes
    '../ui.apps/src/main/content/jcr_root/apps/mysite/**/*.xml',
  ],
  // Prefix to avoid conflicts with Core Component BEM classes
  prefix: 'tw-',
  theme: {
    extend: {
      // Match AEM responsive grid breakpoints
      screens: {
        sm: '576px',
        md: '768px',
        lg: '1024px',
        xl: '1200px',
      },
      colors: {
        primary: {
          DEFAULT: 'var(--color-primary, #0065ff)',
          dark: 'var(--color-primary-dark, #0050cc)',
        },
        secondary: 'var(--color-secondary, #6b7280)',
        accent: 'var(--color-accent, #f59e0b)',
      },
      fontFamily: {
        base: ['var(--font-family-base)', 'system-ui', 'sans-serif'],
        heading: ['var(--font-family-heading)', 'system-ui', 'sans-serif'],
      },
      spacing: {
        // Align with AEM's 8px grid
        '18': '4.5rem',    // 72px (nav height)
        '14': '3.5rem',    // 56px (mobile nav)
      },
    },
  },
  plugins: [
    require('@tailwindcss/typography'),  // Prose styling for RTE output
    require('@tailwindcss/forms'),        // Form element resets
  ],
};
```

---

### 3. CSS Entry Point

```css
/* ui.frontend/src/main.css */
@import "tailwindcss";

/* Custom component layer for AEM-specific styles */
@layer components {
  /* Core Component overrides using Tailwind's @apply */
  .cmp-teaser__title {
    @apply tw-font-heading tw-text-2xl tw-text-gray-900;
  }

  .cmp-teaser__description {
    @apply tw-text-gray-600 tw-leading-relaxed;
  }

  .cmp-teaser__action-link {
    @apply tw-text-primary tw-font-semibold hover:tw-text-primary-dark tw-transition-colors;
  }

  .cmp-button {
    @apply tw-inline-flex tw-items-center tw-px-6 tw-py-3 tw-rounded-lg
           tw-font-semibold tw-transition-all tw-duration-200;
  }

  .cmp-button--primary {
    @apply tw-bg-primary tw-text-white hover:tw-bg-primary-dark;
  }

  .cmp-button--secondary {
    @apply tw-bg-white tw-text-primary tw-border tw-border-primary
           hover:tw-bg-primary hover:tw-text-white;
  }
}

/* Utility layer for project-specific utilities */
@layer utilities {
  .text-balance {
    text-wrap: balance;
  }
}
```

---

### 4. Using Tailwind in HTL

```html
<!-- hero.html -->
<sly data-sly-use.model="com.mysite.models.HeroModel" />
<section class="tw-relative tw-min-h-[500px] tw-flex tw-items-center tw-overflow-hidden">
  <img
    class="tw-absolute tw-inset-0 tw-w-full tw-h-full tw-object-cover"
    src="${model.backgroundImage}"
    alt=""
    loading="eager" />
  <div class="tw-absolute tw-inset-0 tw-bg-black/40"></div>
  <div class="tw-relative tw-z-10 tw-max-w-7xl tw-mx-auto tw-px-6 tw-py-16 md:tw-py-24">
    <h1 class="tw-text-4xl md:tw-text-5xl lg:tw-text-6xl tw-font-heading tw-text-white tw-text-balance">
      ${model.title}
    </h1>
    <p class="tw-mt-4 tw-text-lg tw-text-white/80 tw-max-w-2xl" data-sly-test="${model.subtitle}">
      ${model.subtitle}
    </p>
    <a class="tw-mt-8 tw-inline-block tw-px-8 tw-py-4 tw-bg-white tw-text-primary tw-font-semibold
              tw-rounded-lg hover:tw-bg-gray-100 tw-transition-colors"
      href="${model.ctaLink}"
      data-sly-test="${model.ctaLabel}">
      ${model.ctaLabel}
    </a>
  </div>
</section>
```

**Card grid:**

```html
<!-- card-grid.html -->
<sly data-sly-use.model="com.mysite.models.CardGridModel" />
<div class="tw-grid tw-grid-cols-1 sm:tw-grid-cols-2 lg:tw-grid-cols-3 tw-gap-6 tw-max-w-7xl tw-mx-auto tw-px-6">
  <sly data-sly-list.card="${model.cards}">
    <article class="tw-group tw-rounded-xl tw-overflow-hidden tw-shadow-md hover:tw-shadow-xl tw-transition-shadow tw-bg-white">
      <div class="tw-aspect-video tw-overflow-hidden">
        <img
          class="tw-w-full tw-h-full tw-object-cover group-hover:tw-scale-105 tw-transition-transform tw-duration-300"
          src="${card.image}"
          alt="${card.imageAlt}"
          loading="lazy" />
      </div>
      <div class="tw-p-6">
        <h3 class="tw-text-xl tw-font-heading tw-text-gray-900">${card.title}</h3>
        <p class="tw-mt-2 tw-text-gray-600 tw-line-clamp-3">${card.description}</p>
        <a class="tw-mt-4 tw-inline-flex tw-items-center tw-text-primary tw-font-semibold hover:tw-text-primary-dark"
          href="${card.link}">
          Read More
          <svg class="tw-ml-1 tw-w-4 tw-h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
          </svg>
        </a>
      </div>
    </article>
  </sly>
</div>
```

---

### 5. Prefix Strategy — Tailwind + Core Components

The `prefix: 'tw-'` in the config prevents collisions with Core Component BEM classes:

```html
<!-- Core Component BEM class + Tailwind utility — no conflicts -->
<div class="cmp-teaser tw-rounded-lg tw-shadow-md tw-overflow-hidden">
  <div class="cmp-teaser__image tw-aspect-video">
    <img class="cmp-image__image tw-w-full tw-h-full tw-object-cover" ... />
  </div>
  <div class="cmp-teaser__content tw-p-6">
    <h2 class="cmp-teaser__title tw-text-2xl tw-font-heading">${model.title}</h2>
  </div>
</div>
```

**When to use BEM vs Tailwind:**

| Pattern | Use |
|---------|-----|
| Core Component styling | BEM classes (`.cmp-teaser__title`) via `@apply` in CSS |
| Custom component layout | Tailwind utilities in HTL |
| Spacing, colors, typography | Tailwind utilities |
| Complex states (active tab, expanded accordion) | BEM modifiers (`--active`) + Tailwind |
| Style System variations | BEM modifiers applied by AEM policies |

---

### 6. Tailwind + Style System

AEM Style System adds CSS classes via author dialog. Define Tailwind-based style classes:

```xml
<!-- Content policy for Hero component -->
<styleGroups jcr:primaryType="nt:unstructured">
  <layout jcr:primaryType="nt:unstructured"
    cq:styleGroupLabel="Layout">
    <full-width jcr:primaryType="nt:unstructured"
      cq:styleId="full-width"
      cq:styleLabel="Full Width"
      cq:styleClasses="tw-w-full tw-max-w-none"/>
    <contained jcr:primaryType="nt:unstructured"
      cq:styleId="contained"
      cq:styleLabel="Contained"
      cq:styleClasses="tw-max-w-7xl tw-mx-auto"/>
  </layout>
  <theme jcr:primaryType="nt:unstructured"
    cq:styleGroupLabel="Theme">
    <dark jcr:primaryType="nt:unstructured"
      cq:styleId="dark"
      cq:styleLabel="Dark Background"
      cq:styleClasses="tw-bg-gray-900 tw-text-white"/>
    <light jcr:primaryType="nt:unstructured"
      cq:styleId="light"
      cq:styleLabel="Light Background"
      cq:styleClasses="tw-bg-white tw-text-gray-900"/>
  </theme>
</styleGroups>
```

**Important**: Add Style System class patterns to `tailwind.config.js` content scanning so they're not purged:

```javascript
// tailwind.config.js
module.exports = {
  content: [
    // ... existing paths
    // Scan policy XML for Style System Tailwind classes
    '../ui.apps/src/main/content/jcr_root/conf/mysite/**/*.xml',
  ],
  // Or safelist specific patterns
  safelist: [
    'tw-bg-gray-900', 'tw-text-white',
    'tw-bg-white', 'tw-text-gray-900',
    'tw-max-w-7xl', 'tw-mx-auto',
    'tw-w-full', 'tw-max-w-none',
  ],
};
```

---

### 7. Tailwind Typography for RTE Content

Core Components' RTE outputs raw HTML. Use `@tailwindcss/typography` to style it:

```html
<!-- text.html — Core Component proxy -->
<div class="cmp-text tw-prose tw-prose-lg tw-max-w-none
            tw-prose-headings:tw-font-heading
            tw-prose-a:tw-text-primary
            tw-prose-a:hover:tw-text-primary-dark">
  <sly data-sly-resource="${resource}" />
</div>
```

---

### 8. CSS Custom Properties Bridge

Connect Tailwind's theme to AEM's design tokens via CSS custom properties:

```css
/* ui.frontend/src/main.css */
:root {
  --color-primary: #0065ff;
  --color-primary-dark: #0050cc;
  --color-secondary: #6b7280;
  --color-accent: #f59e0b;
  --font-family-base: 'Inter', system-ui, sans-serif;
  --font-family-heading: 'Inter', system-ui, sans-serif;
}

/* Multi-tenant override */
.brand-a {
  --color-primary: #e63946;
  --color-primary-dark: #c62e3b;
  --font-family-heading: 'Playfair Display', serif;
}
```

Tailwind config references these variables (see section 2), so changing the CSS custom properties re-themes the entire site.

---

### 9. Production Build Optimization

```bash
# Production build — Tailwind JIT only includes used classes
NODE_ENV=production npx webpack --config webpack.prod.js
```

**Expected output:**

```
# Without purging: ~3.5MB CSS
# With JIT (content scanning): ~8-15KB CSS (only used utilities)
```

Verify no unused CSS ships:

```javascript
// webpack.prod.js
const { PurgeCSSPlugin } = require('purgecss-webpack-plugin');

// NOT needed with Tailwind v3+ JIT — it only generates used classes
// Only add PurgeCSS if you have non-Tailwind CSS that needs pruning
```

---

### Anti-Patterns

- **No prefix** — without `prefix: 'tw-'`, Tailwind utilities clash with Core Component BEM classes (`.text-*`, `.bg-*`, `.p-*`)
- **Not scanning HTL files** — Tailwind purges classes it can't find; HTL templates MUST be in the `content` array
- **Using `@apply` everywhere** — defeats the purpose of utility-first; use `@apply` only for Core Component overrides and repeated patterns
- **Style System classes not safelisted** — author-applied classes get purged because they're not in scanned templates
- **Tailwind + SCSS variables for the same values** — pick one source of truth; use CSS custom properties to bridge both
- **Loading full Tailwind in dev, JIT in prod** — always use JIT mode (default in v3+) for consistent behavior
- **Tailwind for Core Component internal styling** — BEM classes are the stable API; use Tailwind for layout, spacing, and custom components
- **Not using `tw-prose` for RTE content** — unstyled RTE output is unreadable
