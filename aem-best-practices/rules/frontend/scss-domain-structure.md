---
title: SCSS Domain-Driven Structure for AEM
impact: HIGH
impactDescription: Flat SCSS folders become unmaintainable at scale — domain-driven organization aligns styles with component ownership and enables selective compilation
tags: scss, css, architecture, domain-driven, structure, aem, ui-frontend, clientlibs
---

## SCSS Domain-Driven Structure for AEM

Organize SCSS in AEM's `ui.frontend` by domain/feature instead of file type. Each domain owns its components, variables, mixins, and overrides — matching the component-driven nature of AEM development.

---

### 1. Domain-Driven Folder Structure

**Correct — organized by domain, variables centralized globally:**

```
ui.frontend/src/
├── main.scss                              # Entry point — imports everything
├── _variables.scss                        # ALL variables in one place (tokens + component vars)
├── _mixins.scss                           # ALL mixins in one place
├── base/                                  # Reset, typography, global styles
│   ├── _reset.scss
│   ├── _typography.scss
│   └── _utilities.scss
├── domains/                               # Feature/domain-scoped styles
│   ├── navigation/
│   │   ├── _index.scss                    # Domain barrel — imports all partials
│   │   ├── _mega-menu.scss
│   │   ├── _mobile-nav.scss
│   │   └── _breadcrumb.scss
│   ├── hero/
│   │   ├── _index.scss
│   │   ├── _hero-banner.scss
│   │   └── _hero-video.scss
│   ├── search/
│   │   ├── _index.scss
│   │   ├── _search-bar.scss
│   │   ├── _search-results.scss
│   │   └── _search-facets.scss
│   ├── product/
│   │   ├── _index.scss
│   │   ├── _product-card.scss
│   │   ├── _product-detail.scss
│   │   └── _product-gallery.scss
│   ├── content/
│   │   ├── _index.scss
│   │   ├── _text.scss
│   │   ├── _accordion.scss
│   │   ├── _tabs.scss
│   │   ├── _carousel.scss
│   │   └── _teaser.scss
│   ├── forms/
│   │   ├── _index.scss
│   │   ├── _contact-form.scss
│   │   └── _newsletter.scss
│   └── footer/
│       ├── _index.scss
│       ├── _footer-links.scss
│       └── _footer-social.scss
├── layout/                                # Global layout patterns
│   ├── _grid.scss
│   ├── _container.scss
│   └── _responsive.scss
└── vendor/                                # Third-party overrides
    ├── _coral-overrides.scss
    └── _core-components.scss
```

**Incorrect — file-type-driven (don't do this):**

```
ui.frontend/src/
├── variables/           # Variables scattered across 30+ files
│   ├── _nav-vars.scss
│   ├── _hero-vars.scss
│   └── ... (hard to find what's defined where)
├── mixins/              # All mixins mixed together
├── components/          # 80+ component files, no ownership
│   ├── _hero.scss
│   ├── _mega-menu.scss
│   ├── _search.scss
│   └── ... (80 more)
├── pages/               # Page-specific overrides (brittle)
└── utilities/           # Everything else
```

---

### 2. Entry Point (main.scss)

```scss
// main.scss — single entry point for Webpack
// Order matters: variables/mixins → base → layout → domains → vendor

// 1. Global variables & mixins (no output — definitions only)
@use 'variables' as *;
@use 'mixins' as *;

// 2. Base styles (reset, typography)
@use 'base/reset';
@use 'base/typography';
@use 'base/utilities';

// 3. Layout (grid, containers)
@use 'layout/grid';
@use 'layout/container';
@use 'layout/responsive';

// 4. Domains (feature-scoped styles)
@use 'domains/navigation';
@use 'domains/hero';
@use 'domains/content';
@use 'domains/search';
@use 'domains/product';
@use 'domains/forms';
@use 'domains/footer';

// 5. Vendor overrides (last — highest specificity)
@use 'vendor/core-components';
@use 'vendor/coral-overrides';
```

---

### 3. Global Variables (`_variables.scss`)

All variables live in one file — design tokens and component-specific values together. One place to look, one place to change.

```scss
// _variables.scss — single source of truth for ALL variables
// Organized by category, component vars grouped under their section

// ============================================================
// DESIGN TOKENS — foundational values
// ============================================================

// Colors
$color-primary: #0065ff;
$color-primary-dark: #0050cc;
$color-secondary: #6b7280;
$color-accent: #f59e0b;
$color-error: #ef4444;
$color-success: #10b981;

$color-text: #1f2937;
$color-text-light: #6b7280;
$color-bg: #ffffff;
$color-bg-alt: #f9fafb;
$color-border: #e5e7eb;

// Typography
$font-family-base: 'Inter', system-ui, -apple-system, sans-serif;
$font-family-heading: 'Inter', system-ui, -apple-system, sans-serif;
$font-family-mono: 'JetBrains Mono', ui-monospace, monospace;

$font-size-xs: 0.75rem;    // 12px
$font-size-sm: 0.875rem;   // 14px
$font-size-base: 1rem;     // 16px
$font-size-lg: 1.125rem;   // 18px
$font-size-xl: 1.25rem;    // 20px
$font-size-2xl: 1.5rem;    // 24px
$font-size-3xl: 1.875rem;  // 30px
$font-size-4xl: 2.25rem;   // 36px

// Spacing (8px grid)
$space-1: 0.25rem;   // 4px
$space-2: 0.5rem;    // 8px
$space-3: 0.75rem;   // 12px
$space-4: 1rem;      // 16px
$space-6: 1.5rem;    // 24px
$space-8: 2rem;      // 32px
$space-12: 3rem;     // 48px
$space-16: 4rem;     // 64px

// Breakpoints (match AEM responsive grid)
$breakpoint-sm: 576px;
$breakpoint-md: 768px;
$breakpoint-lg: 1024px;
$breakpoint-xl: 1200px;

// Borders
$radius-sm: 4px;
$radius-md: 8px;
$radius-lg: 12px;
$radius-full: 9999px;

// Shadows
$shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
$shadow-md: 0 4px 6px rgba(0, 0, 0, 0.07);
$shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.1);

// Transitions
$transition-fast: 150ms ease;
$transition-base: 250ms ease;
$transition-slow: 400ms ease;

// Z-index scale
$z-dropdown: 100;
$z-sticky: 200;
$z-modal-backdrop: 300;
$z-modal: 400;
$z-tooltip: 500;

// ============================================================
// COMPONENT VARIABLES — derived from tokens, grouped by domain
// ============================================================

// Navigation
$nav-height: 72px;
$nav-height-mobile: 56px;
$nav-dropdown-width: 280px;
$nav-bg: $color-bg;
$nav-border: $color-border;

// Hero
$hero-min-height: 500px;
$hero-min-height-mobile: 300px;
$hero-overlay-bg: rgba($color-primary-dark, 0.6);
$hero-title-size: $font-size-4xl;
$hero-title-size-mobile: $font-size-2xl;

// Search
$search-input-height: 48px;
$search-input-radius: $radius-full;
$search-result-gap: $space-4;

// Product
$product-card-radius: $radius-md;
$product-card-shadow: $shadow-md;
$product-card-hover-shadow: $shadow-lg;
$product-gallery-thumb-size: 80px;

// Footer
$footer-bg: $color-text;
$footer-color: $color-bg;
$footer-link-color: rgba($color-bg, 0.7);
$footer-padding: $space-16;

// Modal
$modal-width: 600px;
$modal-width-lg: 900px;
$modal-radius: $radius-lg;
$modal-backdrop-bg: rgba(0, 0, 0, 0.5);
```

**Key rule**: Every variable lives here. When you need a new value for a component, add it to the appropriate section in `_variables.scss`. Never create per-domain `_variables.scss` files — it fragments the system and makes it hard to find what's defined where.

---

### 4. Domain Barrel File

Each domain has an `_index.scss` that imports the global variables and its own partials:

```scss
// domains/navigation/_index.scss
@use 'mega-menu';
@use 'mobile-nav';
@use 'breadcrumb';
```

```scss
// domains/navigation/_mega-menu.scss
@use '../../variables' as *;
@use '../../mixins' as *;

.cmp-navigation {
  height: $nav-height;
  background: $nav-bg;
  border-bottom: 1px solid $nav-border;
  transition: $transition-base;

  &__group {
    display: flex;
    gap: $space-6;
    list-style: none;
  }

  &__item {
    position: relative;

    &-link {
      display: flex;
      align-items: center;
      height: $nav-height;
      padding: 0 $space-4;
      color: $color-text;
      text-decoration: none;
      font-size: $font-size-base;
      transition: color $transition-fast;

      &:hover,
      &:focus-visible {
        color: $color-primary;
      }
    }
  }

  // Mega menu dropdown
  &__dropdown {
    position: absolute;
    top: 100%;
    left: 0;
    min-width: $nav-dropdown-width;
    padding: $space-6;
    background: $color-bg;
    border-radius: $radius-md;
    box-shadow: $shadow-lg;
    opacity: 0;
    visibility: hidden;
    transform: translateY(-8px);
    transition: opacity $transition-base, transform $transition-base, visibility $transition-base;

    .cmp-navigation__item:hover &,
    .cmp-navigation__item:focus-within & {
      opacity: 1;
      visibility: visible;
      transform: translateY(0);
    }
  }

  // Mobile
  @media (max-width: $breakpoint-md) {
    height: $nav-height-mobile;
  }
}
```

---

### 5. Why Global Variables

All variables in `_variables.scss` — no per-domain variable files.

| Benefit | Explanation |
|---------|-------------|
| Single source of truth | One file to search, one file to update |
| Easy theming | Override one file for a different brand |
| No import chains | Domain partials just `@use '../../variables'` |
| Discoverability | New developers find everything in one place |
| Consistency | Component vars derive from tokens in the same file — impossible to drift |

```scss
// Correct — all in _variables.scss
// domains/hero/_hero-banner.scss
@use '../../variables' as *;

.cmp-hero {
  min-height: $hero-min-height;
  background: $hero-overlay-bg;

  &__title {
    font-size: $hero-title-size;
  }
}

// Incorrect — hardcoded values in domain partials
.cmp-hero {
  min-height: 500px;                        // magic number!
  background: rgba(0, 80, 204, 0.6);       // not from variables!
}

// Incorrect — per-domain variable files
// domains/hero/_variables.scss   ← DON'T DO THIS
// domains/search/_variables.scss ← variables scattered everywhere
```

---

### 6. Core Component Override Pattern

```scss
// vendor/_core-components.scss
// Override Core Component BEM classes using global variables
@use '../variables' as *;

// Teaser overrides
.cmp-teaser {
  &__title {
    font-family: $font-family-heading;
    font-size: $font-size-2xl;
    color: $color-text;
  }

  &__description {
    color: $color-text-light;
    line-height: 1.6;
  }

  &__action-link {
    color: $color-primary;
    font-weight: 600;

    &:hover {
      color: $color-primary-dark;
    }
  }
}

// Image overrides
.cmp-image {
  &__image {
    border-radius: $radius-md;
  }
}
```

---

### 7. Responsive Mixins

```scss
// _mixins.scss — all mixins in one file
@use 'variables' as *;

@mixin respond-to($breakpoint) {
  @if $breakpoint == sm { @media (min-width: $breakpoint-sm) { @content; } }
  @else if $breakpoint == md { @media (min-width: $breakpoint-md) { @content; } }
  @else if $breakpoint == lg { @media (min-width: $breakpoint-lg) { @content; } }
  @else if $breakpoint == xl { @media (min-width: $breakpoint-xl) { @content; } }
}

@mixin visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

@mixin focus-ring {
  outline: 2px solid $color-primary;
  outline-offset: 2px;
}

// Usage in domain:
// domains/hero/_hero-banner.scss
@use '../../variables' as *;
@use '../../mixins' as *;

.cmp-hero {
  min-height: 400px;

  @include respond-to(md) {
    min-height: 500px;
  }

  @include respond-to(lg) {
    min-height: 600px;
  }

  &__title {
    font-size: $font-size-2xl;

    @include respond-to(md) {
      font-size: $font-size-3xl;
    }

    @include respond-to(lg) {
      font-size: $font-size-4xl;
    }
  }
}
```

---

### 8. Multi-Tenant Theming

Override tokens per brand/site using Webpack entry points:

```scss
// themes/brand-a/_variables-override.scss
$color-primary: #e63946;
$color-primary-dark: #c62e3b;
$font-family-heading: 'Playfair Display', serif;
$nav-bg: #1a1a2e;
$footer-bg: #1a1a2e;

// themes/brand-a/main.scss
@use 'variables-override' as *;
@use '../../main';   // imports everything with overridden variables
```

```javascript
// webpack.common.js — multi-entry for multi-tenant
module.exports = {
  entry: {
    'clientlib-site': './src/main.scss',
    'clientlib-brand-a': './src/themes/brand-a/main.scss',
    'clientlib-brand-b': './src/themes/brand-b/main.scss',
  },
};
```

---

### 9. Webpack Integration

```javascript
// webpack.common.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.scss$/,
        use: [
          MiniCssExtractPlugin.loader,
          {
            loader: 'css-loader',
            options: { sourceMap: true },
          },
          {
            loader: 'postcss-loader',
            options: {
              postcssOptions: {
                plugins: [['autoprefixer'], ['cssnano', { preset: 'default' }]],
              },
            },
          },
          {
            loader: 'sass-loader',
            options: {
              sassOptions: {
                // Enable @use module system (default in Dart Sass)
                silenceDeprecations: ['legacy-js-api'],
              },
            },
          },
        ],
      },
    ],
  },
};
```

---

### Anti-Patterns

- **Flat `components/` dump** — 80+ SCSS partials in one folder with no domain ownership
- **`@import` instead of `@use`** — `@import` leaks variables globally and is deprecated; always use `@use`
- **Hardcoded values** — colors, spacing, and fonts must come from `_tokens.scss`
- **Page-specific stylesheets** — `_homepage.scss`, `_about.scss` create unmaintainable overrides; style components, not pages
- **Deep nesting (>3 levels)** — `.cmp-hero .cmp-hero__content .cmp-hero__title a span` kills specificity; use BEM flat selectors
- **`!important`** — indicates a specificity problem; fix the cascade instead
- **Per-domain `_variables.scss` files** — scatters variables across 10+ files; keep everything in the global `_variables.scss`
- **Component variables that don't derive from tokens** — `$nav-bg` should reference `$color-bg`, not a hardcoded `#fff`
- **Mixing `@use` and `@import`** — pick `@use` for everything; mixing causes load-order bugs
- **No barrel files** — importing individual partials from outside the domain breaks encapsulation
