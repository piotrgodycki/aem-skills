---
title: Vanilla JS Module Architecture for AEM
impact: HIGH
impactDescription: Proper module organization prevents global scope pollution, enables code splitting, and makes AEM ClientLibs maintainable at scale
tags: vanilla-js, es-modules, dynamic-import, clientlibs, architecture, barrel-exports, dependency-injection
---

## Vanilla JS Module Architecture for AEM

Organize vanilla JavaScript for AEM using ES modules, dynamic imports for code splitting, and a clear dependency structure that works within the AEM ClientLib pipeline.

---

### 1. Module Organization

```
ui.frontend/src/
├── main.ts                        # ClientLib entry point
├── components/                    # Component modules (1:1 with AEM components)
│   ├── accordion/
│   │   ├── index.ts               # barrel export
│   │   ├── accordion.ts           # main module
│   │   ├── accordion.css          # scoped styles
│   │   └── accordion.test.ts
│   ├── tabs/
│   │   ├── index.ts
│   │   ├── tabs.ts
│   │   └── tabs.css
│   └── navigation/
│       ├── index.ts
│       ├── mega-menu.ts
│       ├── mobile-nav.ts
│       └── navigation.css
├── services/                      # Data/API layer
│   ├── graphql.ts
│   ├── search.ts
│   └── analytics.ts
├── utils/                         # Pure utility functions
│   ├── dom.ts
│   ├── events.ts
│   └── storage.ts
└── types/                         # TypeScript type definitions
    └── aem.d.ts
```

---

### 2. Component Module Pattern

```typescript
// components/accordion/accordion.ts

interface AccordionConfig {
  singleOpen?: boolean;
  animationDuration?: number;
}

const DEFAULTS: AccordionConfig = {
  singleOpen: true,
  animationDuration: 300,
};

export function initAccordion(
  root: HTMLElement,
  config: Partial<AccordionConfig> = {}
) {
  const opts = { ...DEFAULTS, ...config };
  const items = root.querySelectorAll<HTMLElement>('.cmp-accordion__item');
  const controller = new AbortController();

  items.forEach(item => {
    const header = item.querySelector<HTMLElement>('.cmp-accordion__header');
    const panel = item.querySelector<HTMLElement>('.cmp-accordion__panel');
    if (!header || !panel) return;

    header.addEventListener('click', () => {
      if (opts.singleOpen) {
        items.forEach(other => {
          if (other !== item) collapseItem(other);
        });
      }
      toggleItem(item);
    }, { signal: controller.signal });
  });

  // Return cleanup function
  return () => controller.abort();
}

function toggleItem(item: HTMLElement) {
  const expanded = item.getAttribute('aria-expanded') === 'true';
  item.setAttribute('aria-expanded', String(!expanded));
  const panel = item.querySelector<HTMLElement>('.cmp-accordion__panel');
  if (panel) panel.hidden = expanded;
}

function collapseItem(item: HTMLElement) {
  item.setAttribute('aria-expanded', 'false');
  const panel = item.querySelector<HTMLElement>('.cmp-accordion__panel');
  if (panel) panel.hidden = true;
}

// components/accordion/index.ts
export { initAccordion } from './accordion';
```

---

### 3. Entry Point with Auto-Init

```typescript
// main.ts — ClientLib entry point
import { initComponentsOnReady } from './utils/init';

// Eager components (above the fold)
import { initAccordion } from './components/accordion';
import { initTabs } from './components/tabs';

// Register components for auto-initialization
const COMPONENTS = {
  '.cmp-accordion': initAccordion,
  '.cmp-tabs': initTabs,
} as const;

// Lazy components (loaded on demand)
const LAZY_COMPONENTS = {
  '.cmp-search': () => import('./components/search').then(m => m.initSearch),
  '.cmp-mega-menu': () => import('./components/navigation').then(m => m.initMegaMenu),
  '.cmp-carousel': () => import('./components/carousel').then(m => m.initCarousel),
} as const;

// Initialize
initComponentsOnReady(COMPONENTS, LAZY_COMPONENTS);
```

```typescript
// utils/init.ts
type InitFn = (root: HTMLElement) => (() => void) | void;
type LazyInitFn = () => Promise<InitFn>;

const cleanups = new Map<HTMLElement, () => void>();

export function initComponentsOnReady(
  eager: Record<string, InitFn>,
  lazy: Record<string, LazyInitFn>
) {
  // Eager: init immediately on DOMContentLoaded
  document.addEventListener('DOMContentLoaded', () => {
    for (const [selector, init] of Object.entries(eager)) {
      document.querySelectorAll<HTMLElement>(selector).forEach(el => {
        const cleanup = init(el);
        if (cleanup) cleanups.set(el, cleanup);
      });
    }

    // Lazy: observe for visibility
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(async (entry) => {
          if (!entry.isIntersecting) return;
          const el = entry.target as HTMLElement;
          observer.unobserve(el);

          for (const [selector, loader] of Object.entries(lazy)) {
            if (el.matches(selector)) {
              const init = await loader();
              const cleanup = init(el);
              if (cleanup) cleanups.set(el, cleanup);
              break;
            }
          }
        });
      },
      { rootMargin: '200px' }
    );

    for (const selector of Object.keys(lazy)) {
      document.querySelectorAll<HTMLElement>(selector).forEach(el => {
        observer.observe(el);
      });
    }
  });
}
```

---

### 4. Dynamic Import for Code Splitting

Webpack splits lazy components into separate chunks:

```javascript
// webpack.prod.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'async',
      minSize: 5000,
      cacheGroups: {
        components: {
          test: /[\\/]components[\\/]/,
          name(module) {
            const match = module.context.match(/[\\/]components[\\/](\w+)/);
            return match ? `component-${match[1]}` : 'components';
          },
        },
      },
    },
  },
};
```

---

### 5. Service Layer (Dependency Injection Lite)

```typescript
// services/graphql.ts
export interface GraphQLService {
  query<T>(persistedQueryPath: string, variables?: Record<string, unknown>): Promise<T>;
}

export function createGraphQLService(endpoint: string): GraphQLService {
  return {
    async query<T>(path: string, variables?: Record<string, unknown>): Promise<T> {
      const params = variables
        ? `;${Object.entries(variables).map(([k, v]) => `${k}=${encodeURIComponent(String(v))}`).join(';')}`
        : '';

      const response = await fetch(`${endpoint}/${path}${params}`);
      if (!response.ok) throw new Error(`GraphQL error: ${response.status}`);
      const json = await response.json();
      if (json.errors?.length) throw new Error(json.errors[0].message);
      return json.data as T;
    },
  };
}

// Singleton instance
export const graphql = createGraphQLService('/graphql/execute.json');

// Usage in component:
import { graphql } from '@services/graphql';

export async function initSearch(root: HTMLElement) {
  const input = root.querySelector<HTMLInputElement>('.cmp-search__input');
  if (!input) return;

  input.addEventListener('input', debounce(async () => {
    const results = await graphql.query('mysite/search', { q: input.value });
    renderResults(root, results);
  }, 300));
}
```

---

### 6. AEM Author Mode Re-Initialization

Handle component re-init when authors add/edit components in the page editor:

```typescript
// utils/author-init.ts
export function observeAuthorChanges(
  initFn: (root: HTMLElement, selector: string) => void
) {
  // Only in author mode
  if (!document.querySelector('meta[name="cq:wcmmode"][content="EDIT"]')) return;

  const observer = new MutationObserver((mutations) => {
    mutations.forEach(mutation => {
      mutation.addedNodes.forEach(node => {
        if (node instanceof HTMLElement) {
          // Re-initialize components in the newly added subtree
          initFn(node, '[data-component]');
        }
      });
    });
  });

  observer.observe(document.body, {
    childList: true,
    subtree: true,
  });
}
```

---

### Anti-Patterns

- **Global functions on `window`** — use ES modules; `window.initAccordion = ...` pollutes global scope
- **Not returning cleanup functions** — causes memory leaks when AEM author mode removes components
- **Importing everything eagerly** — use dynamic `import()` for below-the-fold components
- **Mixing jQuery and vanilla** — pick one; AEM Core Components have moved to vanilla JS
- **Circular dependencies** — components should depend on `utils/` and `services/`, never on each other
- **No AbortController** — event listeners without cleanup leak when components are removed from DOM
