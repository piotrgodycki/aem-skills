---
title: Event Architecture for Vanilla JS AEM
impact: HIGH
impactDescription: Proper event patterns enable decoupled component communication, clean ACDL integration, and memory-safe AEM author mode handling
tags: vanilla-js, events, custom-events, event-delegation, abort-controller, pub-sub, acdl
---

## Event Architecture for Vanilla JS AEM

Event patterns for AEM vanilla JS components — typed custom events, event delegation, AbortController lifecycle, cross-component pub/sub, and Adobe Client Data Layer integration.

---

### 1. Typed Custom Events

```typescript
// utils/events.ts

// Define event types for your site
interface SiteEventMap {
  'site:search-submit': { query: string; facets: Record<string, string[]> };
  'site:cart-update': { items: CartItem[]; total: number };
  'site:modal-open': { id: string; trigger: HTMLElement };
  'site:modal-close': { id: string };
  'site:tab-change': { tabId: string; index: number };
  'site:consent-update': { analytics: boolean; marketing: boolean };
}

export function emitEvent<K extends keyof SiteEventMap>(
  type: K,
  detail: SiteEventMap[K],
  target: EventTarget = document
): void {
  target.dispatchEvent(
    new CustomEvent(type, { detail, bubbles: true, composed: true })
  );
}

export function onEvent<K extends keyof SiteEventMap>(
  type: K,
  handler: (detail: SiteEventMap[K], event: CustomEvent<SiteEventMap[K]>) => void,
  options?: { target?: EventTarget; signal?: AbortSignal }
): void {
  const target = options?.target ?? document;
  target.addEventListener(
    type,
    ((e: CustomEvent<SiteEventMap[K]>) => handler(e.detail, e)) as EventListener,
    { signal: options?.signal }
  );
}

// Usage:
// Emitting
emitEvent('site:search-submit', { query: 'hiking boots', facets: {} });

// Listening
const controller = new AbortController();
onEvent('site:cart-update', ({ items, total }) => {
  updateCartBadge(items.length, total);
}, { signal: controller.signal });

// Cleanup
controller.abort();
```

---

### 2. Event Delegation Utility

```typescript
// utils/delegate.ts
export function delegate<K extends keyof HTMLElementEventMap>(
  root: HTMLElement,
  eventType: K,
  selector: string,
  handler: (event: HTMLElementEventMap[K], target: HTMLElement) => void,
  options?: { signal?: AbortSignal }
): void {
  root.addEventListener(
    eventType,
    (event) => {
      const target = (event.target as HTMLElement).closest<HTMLElement>(selector);
      if (target && root.contains(target)) {
        handler(event, target);
      }
    },
    { signal: options?.signal }
  );
}

// Usage — single listener for N cards:
export function initCardGrid(root: HTMLElement) {
  const controller = new AbortController();

  delegate(root, 'click', '.cmp-card__link', (event, target) => {
    // Track click
    emitEvent('site:card-click', {
      title: target.textContent || '',
      href: target.getAttribute('href') || '',
    });
  }, { signal: controller.signal });

  delegate(root, 'mouseenter', '.cmp-card', (_event, target) => {
    target.classList.add('cmp-card--hover');
  }, { signal: controller.signal });

  delegate(root, 'mouseleave', '.cmp-card', (_event, target) => {
    target.classList.remove('cmp-card--hover');
  }, { signal: controller.signal });

  return () => controller.abort();
}
```

---

### 3. AbortController Lifecycle

```typescript
export function initComponent(root: HTMLElement) {
  // One controller per component instance
  const controller = new AbortController();
  const { signal } = controller;

  // All event listeners use the same signal
  root.addEventListener('click', handleClick, { signal });
  document.addEventListener('keydown', handleEscape, { signal });
  window.addEventListener('resize', handleResize, { signal, passive: true });

  // Custom events too
  onEvent('site:theme-change', applyTheme, { signal });

  // Timers (manual cleanup since they don't support AbortSignal)
  const timer = setInterval(() => {
    if (signal.aborted) { clearInterval(timer); return; }
    poll();
  }, 5000);

  // Single cleanup aborts everything
  return () => {
    controller.abort();
    clearInterval(timer);
  };
}
```

---

### 4. Pub/Sub Event Bus

Decoupled communication between components that don't share a DOM tree:

```typescript
// utils/event-bus.ts
type Handler<T = unknown> = (data: T) => void;

class EventBus {
  private handlers = new Map<string, Set<Handler>>();

  on<T>(event: string, handler: Handler<T>): () => void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set());
    }
    this.handlers.get(event)!.add(handler as Handler);

    // Return unsubscribe function
    return () => this.handlers.get(event)?.delete(handler as Handler);
  }

  once<T>(event: string, handler: Handler<T>): () => void {
    const wrapper: Handler<T> = (data) => {
      handler(data);
      this.handlers.get(event)?.delete(wrapper as Handler);
    };
    return this.on(event, wrapper);
  }

  emit<T>(event: string, data: T): void {
    this.handlers.get(event)?.forEach(handler => {
      try {
        handler(data);
      } catch (err) {
        console.error(`[EventBus] Error in handler for "${event}":`, err);
      }
    });
  }

  off(event: string): void {
    this.handlers.delete(event);
  }
}

export const bus = new EventBus();

// Usage:
// In navigation component:
const unsub = bus.on('cart:updated', (data: { count: number }) => {
  cartBadge.textContent = String(data.count);
});

// In cart component:
bus.emit('cart:updated', { count: items.length });

// Cleanup:
unsub();
```

---

### 5. Adobe Client Data Layer Integration

Push events and data to ACDL from vanilla JS components:

```typescript
// utils/analytics.ts
declare global {
  interface Window {
    adobeDataLayer?: Array<Record<string, unknown>>;
  }
}

export function pushToDataLayer(data: Record<string, unknown>): void {
  window.adobeDataLayer = window.adobeDataLayer || [];
  window.adobeDataLayer.push(data);
}

export function trackEvent(eventName: string, data?: Record<string, unknown>): void {
  pushToDataLayer({
    event: eventName,
    eventInfo: {
      path: window.location.pathname,
      timestamp: new Date().toISOString(),
      ...data,
    },
  });
}

// Component integration:
export function initSearchWithTracking(root: HTMLElement) {
  const controller = new AbortController();

  root.addEventListener('submit', (e) => {
    e.preventDefault();
    const input = root.querySelector<HTMLInputElement>('.cmp-search__input');
    const query = input?.value.trim() || '';

    trackEvent('search:submit', {
      searchTerm: query,
      searchLocation: 'header',
    });

    performSearch(query);
  }, { signal: controller.signal });

  // Listen for site events and bridge to ACDL
  onEvent('site:card-click', ({ title, href }) => {
    trackEvent('cta:click', { ctaText: title, ctaUrl: href });
  }, { signal: controller.signal });

  return () => controller.abort();
}
```

---

### 6. Cross-Component Communication Pattern

```
┌─────────────┐    CustomEvent     ┌─────────────┐
│   Search     │ ─────────────────→│  Results     │
│   Component  │  'site:search'    │  Component   │
└─────────────┘                    └─────────────┘
       │                                  │
       │  bus.emit('search:complete')     │
       ▼                                  ▼
┌─────────────┐                    ┌─────────────┐
│  Analytics   │←── bus.on() ──────│  URL State   │
│  Tracker     │                   │  Manager     │
└─────────────┘                    └─────────────┘
```

**Rules for choosing**:

| Pattern | When to Use |
|---------|------------|
| DOM CustomEvent | Components share a DOM ancestor; need bubbling |
| EventBus (pub/sub) | Components are in different DOM trees |
| Direct function call | One component owns another (parent → child) |
| ACDL push | Analytics/tracking — anything Adobe Launch consumes |

---

### Anti-Patterns

- **Removing event listeners by reference** — use AbortController instead of storing function references
- **Events without cleanup** — every `addEventListener` must have a corresponding abort or removal
- **Synchronous heavy work in event handlers** — debounce/throttle user input events
- **Missing `passive: true`** on scroll/touch listeners — blocks scrolling, hurts INP
- **Global event bus without namespacing** — prefix events by feature (`search:submit`, `cart:update`)
- **Not using `composed: true`** on custom events that need to cross Shadow DOM boundaries
- **Emitting events in constructor** — components may not be listening yet; defer to `connectedCallback` or init
