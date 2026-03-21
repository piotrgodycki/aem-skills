---
title: Lightweight Preact Components for AEM
impact: MEDIUM
impactDescription: Preact islands enable progressive enhancement — interactive features on top of server-rendered HTL without SPA overhead
tags: preact, islands, progressive-enhancement, partial-hydration, htl, lightweight
---

## Lightweight Preact Components for AEM

Build small, focused interactive components that enhance server-rendered HTL pages. The islands architecture mounts Preact only where interactivity is needed — the rest of the page stays static HTML.

---

### 1. Islands Architecture

```
┌─────────────────────────────────────────────────┐
│  HTL-rendered page (static HTML)                │
│                                                 │
│  ┌──────────────┐    ┌──────────────────────┐   │
│  │ Preact Island │    │  Static HTL content  │   │
│  │ (interactive) │    │  (no JS needed)      │   │
│  └──────────────┘    └──────────────────────┘   │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │  Static HTL content                      │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  ┌──────────────┐    ┌──────────────┐           │
│  │ Preact Island │    │ Preact Island│           │
│  │ (tabs)        │    │ (search)     │           │
│  └──────────────┘    └──────────────┘           │
└─────────────────────────────────────────────────┘
```

---

### 2. Island Mounting Utility

```typescript
// shared/utils/island.ts
import { render, h, type ComponentType } from 'preact';

interface IslandOptions {
  lazy?: boolean;
  rootMargin?: string;
}

export function mountIsland<P extends Record<string, unknown>>(
  selector: string,
  Component: ComponentType<P>,
  options: IslandOptions = {}
) {
  const elements = document.querySelectorAll<HTMLElement>(selector);
  if (!elements.length) return;

  elements.forEach(el => {
    const props = parseProps<P>(el);

    if (options.lazy) {
      lazyMount(el, Component, props, options.rootMargin);
    } else {
      hydrate(el, Component, props);
    }
  });
}

function parseProps<P>(el: HTMLElement): P {
  const propsAttr = el.getAttribute('data-props');
  if (propsAttr) {
    try { return JSON.parse(propsAttr); } catch { /* fall through */ }
  }

  // Collect data-* attributes as props
  const props: Record<string, string> = {};
  for (const { name, value } of Array.from(el.attributes)) {
    if (name.startsWith('data-') && name !== 'data-component' && name !== 'data-props') {
      const key = name.slice(5).replace(/-([a-z])/g, (_, c) => c.toUpperCase());
      props[key] = value;
    }
  }
  return props as P;
}

function hydrate<P>(el: HTMLElement, Component: ComponentType<P>, props: P) {
  // Preserve SSR content as children if no explicit props
  render(h(Component, props), el);
}

function lazyMount<P>(
  el: HTMLElement,
  Component: ComponentType<P>,
  props: P,
  rootMargin = '200px'
) {
  const observer = new IntersectionObserver(
    ([entry]) => {
      if (entry.isIntersecting) {
        observer.disconnect();
        hydrate(el, Component, props);
      }
    },
    { rootMargin }
  );
  observer.observe(el);
}

// Usage:
import { Tabs } from '@features/tabs';
import { Accordion } from '@features/accordion';
import { SearchBar } from '@features/search';

// Eager — above the fold
mountIsland('[data-component="search"]', SearchBar);

// Lazy — below the fold
mountIsland('[data-component="tabs"]', Tabs, { lazy: true });
mountIsland('[data-component="accordion"]', Accordion, { lazy: true });
```

---

### 3. Progressive Enhancement: Tabs

The tabs work without JS (all panels visible), then Preact adds interactivity:

**HTL (server-rendered, accessible without JS):**

```html
<sly data-sly-use.model="com.mysite.models.TabsModel" />
<div data-component="tabs" data-props='${model.jsonExport @ context="scriptString"}' class="cmp-tabs">
  <ol class="cmp-tabs__tablist" role="tablist">
    <sly data-sly-list.tab="${model.tabs}">
      <li class="cmp-tabs__tab" role="tab">${tab.title}</li>
    </sly>
  </ol>
  <sly data-sly-list.tab="${model.tabs}">
    <div class="cmp-tabs__tabpanel" role="tabpanel">
      <sly data-sly-resource="${tab.resource}" />
    </div>
  </sly>
</div>
```

**Preact component (enhances the HTL markup):**

```tsx
// features/tabs/Tabs.tsx
import { signal } from '@preact/signals';

interface TabsProps {
  tabs: Array<{ id: string; title: string }>;
}

export function Tabs({ tabs }: TabsProps) {
  const activeIndex = signal(0);

  return (
    <div class="cmp-tabs">
      <ol class="cmp-tabs__tablist" role="tablist">
        {tabs.map((tab, i) => (
          <li
            key={tab.id}
            class={`cmp-tabs__tab ${i === activeIndex.value ? 'cmp-tabs__tab--active' : ''}`}
            role="tab"
            aria-selected={i === activeIndex.value}
            tabIndex={i === activeIndex.value ? 0 : -1}
            onClick={() => { activeIndex.value = i; }}
            onKeyDown={(e) => {
              if (e.key === 'ArrowRight') activeIndex.value = Math.min(i + 1, tabs.length - 1);
              if (e.key === 'ArrowLeft') activeIndex.value = Math.max(i - 1, 0);
            }}
          >
            {tab.title}
          </li>
        ))}
      </ol>
      {tabs.map((tab, i) => (
        <div
          key={tab.id}
          class={`cmp-tabs__tabpanel ${i === activeIndex.value ? 'cmp-tabs__tabpanel--active' : ''}`}
          role="tabpanel"
          hidden={i !== activeIndex.value}
        >
          {/* Content injected via HTL data or fetched */}
        </div>
      ))}
    </div>
  );
}
```

---

### 4. Preact + HTL Hybrid Rendering

For complex cases where Preact manages only a slice of the component:

```html
<!-- hero.html — HTL renders structure, Preact handles only the CTA animation -->
<section class="cmp-hero" style="background-image: url('${model.backgroundImage}')">
  <div class="cmp-hero__content">
    <h1 class="cmp-hero__title">${model.title}</h1>
    <p class="cmp-hero__subtitle">${model.subtitle}</p>
    <!-- Only this part is a Preact island -->
    <div data-component="animated-cta" data-label="${model.ctaLabel}" data-link="${model.ctaLink}"></div>
  </div>
</section>
```

```tsx
// features/hero/AnimatedCta.tsx
import { useRef, useEffect } from 'preact/hooks';

interface AnimatedCtaProps {
  label: string;
  link: string;
}

export function AnimatedCta({ label, link }: AnimatedCtaProps) {
  const ref = useRef<HTMLAnchorElement>(null);

  useEffect(() => {
    if (!ref.current) return;
    ref.current.animate(
      [{ opacity: 0, transform: 'translateY(10px)' }, { opacity: 1, transform: 'translateY(0)' }],
      { duration: 400, easing: 'ease-out', fill: 'forwards' }
    );
  }, []);

  return (
    <a ref={ref} class="cmp-hero__cta" href={link} style={{ opacity: 0 }}>
      {label}
    </a>
  );
}
```

---

### 5. Partial Hydration with Slot-Like Content

Preserve server-rendered child content while adding Preact interactivity:

```typescript
// shared/utils/preserve-children.ts
import { h, type VNode } from 'preact';

export function preserveChildren(el: HTMLElement): VNode {
  return h('div', {
    dangerouslySetInnerHTML: { __html: el.innerHTML },
  });
}

// Usage in island mount:
function mountWithChildren(selector: string) {
  document.querySelectorAll<HTMLElement>(selector).forEach(el => {
    const serverContent = el.innerHTML;
    const props = JSON.parse(el.getAttribute('data-props') || '{}');

    render(
      h(AccordionIsland, {
        ...props,
        serverContent,
      }),
      el
    );
  });
}
```

---

### Anti-Patterns

- **Full Preact SPA for a traditional AEM site** — use islands; most AEM pages are mostly static
- **Mounting islands before DOMContentLoaded** — defer island mounting to avoid blocking render
- **Not providing no-JS fallback** — HTL should render a usable (if less interactive) version
- **Importing entire Preact in each island** — Webpack deduplicates, but ensure no duplicate bundles
- **Using Preact for purely decorative animations** — CSS animations/transitions are more efficient
- **Large island scope** — mount the smallest interactive unit, not entire page sections
