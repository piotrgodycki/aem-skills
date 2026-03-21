---
title: Custom React Hooks for AEM
impact: HIGH
impactDescription: Reusable AEM hooks eliminate boilerplate and ensure consistent handling of content fragments, author mode, and responsive behavior
tags: react, hooks, aem, content-fragment, graphql, author-mode, breakpoint, intersection
---

## Custom React Hooks for AEM

Production-ready hooks that encapsulate common AEM patterns — content fragment fetching, GraphQL queries, author mode detection, responsive breakpoints, and intersection observation.

---

### 1. useContentFragment

Fetch a single Content Fragment by path with caching:

```typescript
// shared/hooks/useContentFragment.ts
import { useState, useEffect } from 'react';

interface UseCFOptions {
  variation?: string;
  elements?: string[];
}

interface UseCFResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

export function useContentFragment<T = Record<string, unknown>>(
  cfPath: string,
  options: UseCFOptions = {}
): UseCFResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    if (!cfPath) {
      setLoading(false);
      return;
    }

    const controller = new AbortController();

    const variation = options.variation ? `.${options.variation}` : '';
    const elements = options.elements?.length
      ? `?elements=${options.elements.join(',')}`
      : '';

    fetch(
      `/api/assets${cfPath}${variation}.json${elements}`,
      { signal: controller.signal }
    )
      .then(res => {
        if (!res.ok) throw new Error(`CF fetch failed: ${res.status}`);
        return res.json();
      })
      .then(json => setData(json.properties?.elements as T))
      .catch(err => {
        if (err.name !== 'AbortError') setError(err);
      })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [cfPath, options.variation, options.elements?.join(',')]);

  return { data, loading, error };
}

// Usage:
interface ArticleCF {
  title: { value: string };
  body: { value: string };
  author: { value: string };
}

function ArticleDetail({ cfPath }: { cfPath: string }) {
  const { data, loading, error } = useContentFragment<ArticleCF>(cfPath);
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return (
    <article>
      <h1>{data?.title.value}</h1>
      <div dangerouslySetInnerHTML={{ __html: data?.body.value || '' }} />
    </article>
  );
}
```

---

### 2. useGraphQL

Execute AEM persisted GraphQL queries:

```typescript
// shared/hooks/useGraphQL.ts
import { useState, useEffect, useCallback } from 'react';

interface UseGQLOptions<V> {
  variables?: V;
  skip?: boolean;
}

interface UseGQLResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

const GQL_ENDPOINT = '/graphql/execute.json';

export function useGraphQL<T = unknown, V = Record<string, unknown>>(
  persistedQueryPath: string,
  options: UseGQLOptions<V> = {}
): UseGQLResult<T> {
  const { variables, skip = false } = options;
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(!skip);
  const [error, setError] = useState<Error | null>(null);
  const [trigger, setTrigger] = useState(0);

  const refetch = useCallback(() => setTrigger(t => t + 1), []);

  useEffect(() => {
    if (skip) return;

    const controller = new AbortController();
    setLoading(true);

    const params = variables
      ? `;${Object.entries(variables).map(([k, v]) => `${k}=${encodeURIComponent(String(v))}`).join(';')}`
      : '';

    fetch(`${GQL_ENDPOINT}/${persistedQueryPath}${params}`, {
      signal: controller.signal,
      headers: { 'Content-Type': 'application/json' },
    })
      .then(res => {
        if (!res.ok) throw new Error(`GraphQL error: ${res.status}`);
        return res.json();
      })
      .then(json => {
        if (json.errors?.length) throw new Error(json.errors[0].message);
        setData(json.data as T);
      })
      .catch(err => {
        if (err.name !== 'AbortError') setError(err);
      })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [persistedQueryPath, JSON.stringify(variables), skip, trigger]);

  return { data, loading, error, refetch };
}

// Usage:
interface ArticleListData {
  articleList: {
    items: Array<{
      _path: string;
      title: string;
      slug: string;
    }>;
  };
}

function ArticleList({ category }: { category: string }) {
  const { data, loading } = useGraphQL<ArticleListData>(
    'mysite/articles-by-category',
    { variables: { category } }
  );

  if (loading) return <Spinner />;
  return (
    <ul>
      {data?.articleList.items.map(a => (
        <li key={a._path}>{a.title}</li>
      ))}
    </ul>
  );
}
```

---

### 3. useAuthorMode

Detect AEM authoring context:

```typescript
// shared/hooks/useAuthorMode.ts
import { useSyncExternalStore } from 'react';

type WcmMode = 'EDIT' | 'PREVIEW' | 'DESIGN' | 'DISABLED';

interface AuthorModeState {
  isAuthorMode: boolean;
  isEditMode: boolean;
  isPreviewMode: boolean;
  wcmMode: WcmMode;
}

function getWcmMode(): WcmMode {
  if (typeof document === 'undefined') return 'DISABLED';
  const meta = document.querySelector('meta[name="cq:wcmmode"]');
  return (meta?.getAttribute('content') as WcmMode) || 'DISABLED';
}

let cachedState: AuthorModeState | null = null;

function getSnapshot(): AuthorModeState {
  if (!cachedState) {
    const wcmMode = getWcmMode();
    cachedState = {
      isAuthorMode: wcmMode !== 'DISABLED',
      isEditMode: wcmMode === 'EDIT',
      isPreviewMode: wcmMode === 'PREVIEW',
      wcmMode,
    };
  }
  return cachedState;
}

function subscribe(callback: () => void): () => void {
  // WCM mode doesn't change during page lifecycle
  return () => {};
}

export function useAuthorMode(): AuthorModeState {
  return useSyncExternalStore(subscribe, getSnapshot, () => ({
    isAuthorMode: false,
    isEditMode: false,
    isPreviewMode: false,
    wcmMode: 'DISABLED' as const,
  }));
}

// Usage:
function HeroBanner({ title, image }: HeroProps) {
  const { isEditMode, isAuthorMode } = useAuthorMode();

  return (
    <div className="cmp-hero">
      {/* Show placeholder in edit mode when empty */}
      {isEditMode && !title && (
        <div className="cmp-hero__placeholder">
          Drag content here or open the dialog
        </div>
      )}
      {/* Skip animations in author mode */}
      <h1 className={isAuthorMode ? '' : 'cmp-hero__title--animated'}>
        {title}
      </h1>
    </div>
  );
}
```

---

### 4. useBreakpoint

Responsive breakpoint detection matching AEM's responsive grid:

```typescript
// shared/hooks/useBreakpoint.ts
import { useSyncExternalStore } from 'react';

type Breakpoint = 'mobile' | 'tablet' | 'desktop';

const BREAKPOINTS: Record<Breakpoint, string> = {
  mobile: '(max-width: 767px)',
  tablet: '(min-width: 768px) and (max-width: 1199px)',
  desktop: '(min-width: 1200px)',
};

function getCurrentBreakpoint(): Breakpoint {
  if (typeof window === 'undefined') return 'desktop';
  if (window.matchMedia(BREAKPOINTS.mobile).matches) return 'mobile';
  if (window.matchMedia(BREAKPOINTS.tablet).matches) return 'tablet';
  return 'desktop';
}

let currentBreakpoint = getCurrentBreakpoint();
const listeners = new Set<() => void>();

if (typeof window !== 'undefined') {
  Object.values(BREAKPOINTS).forEach(query => {
    window.matchMedia(query).addEventListener('change', () => {
      currentBreakpoint = getCurrentBreakpoint();
      listeners.forEach(fn => fn());
    });
  });
}

export function useBreakpoint(): Breakpoint {
  return useSyncExternalStore(
    (cb) => { listeners.add(cb); return () => listeners.delete(cb); },
    () => currentBreakpoint,
    () => 'desktop' as Breakpoint
  );
}

// Usage:
function Navigation() {
  const breakpoint = useBreakpoint();

  return breakpoint === 'mobile'
    ? <MobileNav />
    : <DesktopNav />;
}
```

---

### 5. useIntersection

Lazy-load or animate components when they enter the viewport:

```typescript
// shared/hooks/useIntersection.ts
import { useRef, useState, useEffect, type RefObject } from 'react';

interface UseIntersectionOptions extends IntersectionObserverInit {
  triggerOnce?: boolean;
}

export function useIntersection<T extends HTMLElement = HTMLDivElement>(
  options: UseIntersectionOptions = {}
): [RefObject<T | null>, boolean] {
  const { triggerOnce = false, threshold = 0, rootMargin = '200px', root } = options;
  const ref = useRef<T | null>(null);
  const [isIntersecting, setIntersecting] = useState(false);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        setIntersecting(entry.isIntersecting);
        if (entry.isIntersecting && triggerOnce) {
          observer.unobserve(el);
        }
      },
      { threshold, rootMargin, root }
    );

    observer.observe(el);
    return () => observer.disconnect();
  }, [threshold, rootMargin, triggerOnce, root]);

  return [ref, isIntersecting];
}

// Usage — lazy render:
function LazyVideo({ src }: { src: string }) {
  const [ref, isVisible] = useIntersection<HTMLDivElement>({ triggerOnce: true });

  return (
    <div ref={ref} className="cmp-video">
      {isVisible ? <video src={src} controls /> : <div className="cmp-video__placeholder" />}
    </div>
  );
}
```

---

### 6. useAemPage

Access current AEM page path and model data:

```typescript
// shared/hooks/useAemPage.ts
import { useState, useEffect } from 'react';
import { ModelManager } from '@adobe/aem-spa-page-model-manager';

interface AemPageModel {
  path: string;
  title: string;
  description: string;
  language: string;
  template: string;
  lastModified: number;
}

export function useAemPage(): AemPageModel | null {
  const [page, setPage] = useState<AemPageModel | null>(null);

  useEffect(() => {
    ModelManager.getData().then(model => {
      setPage({
        path: model[':path'] || '',
        title: model['jcr:title'] || '',
        description: model['jcr:description'] || '',
        language: model.language || 'en',
        template: model['cq:template'] || '',
        lastModified: model['jcr:lastModified'] || 0,
      });
    });
  }, []);

  return page;
}
```

---

### 7. Hook Composition

Combine hooks for complex behavior:

```typescript
// features/hero/useHeroAnimation.ts
import { useEffect, useRef } from 'react';
import { useIntersection } from '@shared/hooks/useIntersection';
import { useAuthorMode } from '@shared/hooks/useAuthorMode';
import { useBreakpoint } from '@shared/hooks/useBreakpoint';

export function useHeroAnimation() {
  const { isAuthorMode } = useAuthorMode();
  const breakpoint = useBreakpoint();
  const [ref, isVisible] = useIntersection<HTMLDivElement>({ triggerOnce: true });
  const animationRef = useRef<Animation | null>(null);

  useEffect(() => {
    if (isAuthorMode || breakpoint === 'mobile' || !isVisible) return;

    const el = ref.current;
    if (!el) return;

    animationRef.current = el.animate(
      [
        { opacity: 0, transform: 'translateY(20px)' },
        { opacity: 1, transform: 'translateY(0)' },
      ],
      { duration: 600, easing: 'ease-out', fill: 'forwards' }
    );

    return () => animationRef.current?.cancel();
  }, [isVisible, isAuthorMode, breakpoint]);

  return { ref, isVisible };
}
```

---

### Anti-Patterns

- **Fetching in useEffect without AbortController** — causes race conditions and memory leaks on unmount
- **Not handling author mode** — animations and lazy loading break the page editor
- **Inline fetch URLs** — hardcoded hosts break across environments; use relative paths or config
- **Missing SSR fallbacks** — `useSyncExternalStore` needs a `getServerSnapshot` for SSR
- **Hook dependencies on object references** — `JSON.stringify(variables)` in deps to avoid infinite loops
- **No error boundaries** — a failed hook crashes the entire SPA instead of the affected component
- **Polling in hooks** — use ModelManager events or SWR revalidation instead of `setInterval`
