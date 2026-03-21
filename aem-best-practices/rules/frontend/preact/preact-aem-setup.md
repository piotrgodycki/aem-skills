---
title: Preact Setup for AEM
impact: HIGH
impactDescription: Preact reduces bundle size by ~30KB vs React while maintaining AEM SPA Editor compatibility through the compat layer
tags: preact, aem, webpack, compat, bundle-size, htl, setup
---

## Preact Setup for AEM

Preact is a 3KB alternative to React that works with AEM through the compatibility layer. Use it when bundle size is critical — traditional AEM sites with interactive islands, not full SPA Editor projects.

---

### 1. When to Choose Preact vs React

| Criterion | Preact | React |
|-----------|--------|-------|
| SPA Editor (full page) | No (compat issues with ResponsiveGrid) | Yes |
| Interactive islands in HTL pages | Yes (ideal) | Overkill |
| Bundle size matters | 3KB + app code | ~40KB + app code |
| Core Component JS extension | Yes | No (different rendering model) |
| Headless / Next.js | No (use React) | Yes |
| Team familiarity | Needs learning | Standard |

**Rule**: Use React for SPA Editor projects. Use Preact for traditional AEM (HTL pages) when you need interactive islands.

---

### 2. Webpack Configuration

```javascript
// ui.frontend/webpack.common.js
module.exports = {
  resolve: {
    alias: {
      'react': 'preact/compat',
      'react-dom': 'preact/compat',
      'react-dom/test-utils': 'preact/test-utils',
      'react/jsx-runtime': 'preact/jsx-runtime',
    },
    extensions: ['.tsx', '.ts', '.js', '.jsx'],
  },
  module: {
    rules: [
      {
        test: /\.[tj]sx?$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              ['@babel/preset-env', { targets: '> 0.5%, not dead' }],
              ['@babel/preset-typescript'],
            ],
            plugins: [
              ['@babel/plugin-transform-react-jsx', {
                runtime: 'automatic',
                importSource: 'preact',
              }],
            ],
          },
        },
      },
    ],
  },
};
```

**package.json dependencies:**

```json
{
  "dependencies": {
    "preact": "^10.19.0"
  },
  "devDependencies": {
    "@babel/preset-typescript": "^7.23.0",
    "babel-loader": "^9.1.3"
  }
}
```

---

### 3. HTL Integration (Islands Pattern)

Mount Preact components into HTL-rendered pages:

```html
<!-- hero.html (HTL) -->
<sly data-sly-use.model="com.mysite.models.HeroModel" />
<div
  data-component="hero"
  data-props='${model.jsonExport @ context="scriptString"}'
  class="cmp-hero">
  <!-- SSR fallback / no-JS content -->
  <h1 class="cmp-hero__title">${model.title}</h1>
</div>
```

```typescript
// ui.frontend/src/islands/hero.tsx
import { render, h } from 'preact';
import { Hero } from '@features/hero';

document.querySelectorAll('[data-component="hero"]').forEach(el => {
  const props = JSON.parse(el.getAttribute('data-props') || '{}');
  render(<Hero {...props} />, el);
});
```

**ClientLib entry point:**

```typescript
// ui.frontend/src/main.ts
// Eagerly mount above-the-fold islands
import './islands/hero';

// Lazily mount below-the-fold islands
const observer = new IntersectionObserver((entries) => {
  entries.forEach(async (entry) => {
    if (!entry.isIntersecting) return;
    const el = entry.target as HTMLElement;
    const component = el.dataset.component;
    observer.unobserve(el);

    switch (component) {
      case 'tabs':
        const { mountTabs } = await import('./islands/tabs');
        mountTabs(el);
        break;
      case 'accordion':
        const { mountAccordion } = await import('./islands/accordion');
        mountAccordion(el);
        break;
    }
  });
});

document.querySelectorAll('[data-component][data-lazy]').forEach(el => {
  observer.observe(el);
});
```

---

### 4. Preact + AEM SPA SDK (Compat Layer)

For simpler SPA Editor use cases (not recommended for complex projects):

```typescript
// With preact/compat aliased, AEM SPA SDK works for basic cases:
import { MapTo } from '@adobe/aem-react-editable-components';
import { Hero } from '@features/hero';

// This works through preact/compat
MapTo('mysite/components/hero')(Hero);

// ⚠️ Known limitations with compat:
// - ResponsiveGrid may have rendering issues
// - Some editor overlays don't position correctly
// - React.lazy() works but Suspense has edge cases
```

---

### 5. Bundle Size Comparison

```
# React build
react (45.2 KB) + react-dom (130.1 KB) = ~42 KB gzipped

# Preact build
preact (4.2 KB) + preact/compat (2.1 KB) = ~3 KB gzipped

# Savings: ~39 KB gzipped = faster FCP on 3G networks
```

---

### Anti-Patterns

- **Using Preact with full SPA Editor** — compat layer has edge cases with ResponsiveGrid and editor overlays
- **Not aliasing in Webpack** — importing both React and Preact doubles the bundle
- **Using React-specific features without compat** — `useId`, `useSyncExternalStore` need compat
- **Skipping HTL fallback** — Preact islands should degrade gracefully with JS disabled
- **Mixing Preact and React in the same bundle** — pick one per project
