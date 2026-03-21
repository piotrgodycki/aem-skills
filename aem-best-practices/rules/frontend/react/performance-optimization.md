---
title: React Performance Optimization for AEM
impact: CRITICAL
impactDescription: Unoptimized React components cause unnecessary re-renders, large bundles, and poor Core Web Vitals — directly impacting SEO and user experience
tags: react, performance, memo, useMemo, useCallback, code-splitting, lazy, suspense, transitions, cwv, bundle
---

## React Performance Optimization for AEM

Production performance patterns for AEM React applications — memoization, code splitting, render optimization, bundle analysis, and Core Web Vitals alignment. Based on Vercel React best practices adapted for AEM headless and hybrid architectures.

---

### 1. Component Memoization

Use `React.memo` to prevent re-renders when props haven't changed:

```tsx
import { memo } from 'react';

// Correct — memoize components that receive the same props frequently
interface CardProps {
  title: string;
  image: string;
  link: string;
}

export const Card = memo(function Card({ title, image, link }: CardProps) {
  return (
    <div className="cmp-card">
      <img className="cmp-card__image" src={image} alt="" loading="lazy" />
      <h3 className="cmp-card__title">{title}</h3>
      <a className="cmp-card__link" href={link}>Read More</a>
    </div>
  );
});

// Correct — custom comparison for complex props
interface ProductCardProps {
  product: { sku: string; title: string; price: number; image: string };
  onAddToCart: (sku: string) => void;
}

export const ProductCard = memo(
  function ProductCard({ product, onAddToCart }: ProductCardProps) {
    return (
      <div className="cmp-product-card">
        <img src={product.image} alt={product.title} loading="lazy" />
        <h3>{product.title}</h3>
        <p>${product.price}</p>
        <button onClick={() => onAddToCart(product.sku)}>Add to Cart</button>
      </div>
    );
  },
  (prev, next) => prev.product.sku === next.product.sku && prev.product.price === next.product.price
);
```

**When NOT to memoize**:
- Components that always receive new props (e.g., children, inline objects)
- Simple components with minimal render cost
- Components that render rarely

---

### 2. useMemo — Expensive Computations

```tsx
import { useMemo } from 'react';

function SearchResults({ results, query, facets }: SearchResultsProps) {
  // Correct — expensive filtering/sorting memoized
  const filteredResults = useMemo(() => {
    return results
      .filter(r => matchesFacets(r, facets))
      .sort((a, b) => scoreRelevance(b, query) - scoreRelevance(a, query));
  }, [results, query, facets]);

  // Correct — expensive derived data
  const facetCounts = useMemo(() => {
    const counts: Record<string, number> = {};
    results.forEach(r => {
      r.tags.forEach(tag => {
        counts[tag] = (counts[tag] || 0) + 1;
      });
    });
    return counts;
  }, [results]);

  return (
    <div>
      <FacetPanel counts={facetCounts} />
      <ResultList items={filteredResults} />
    </div>
  );
}

// Incorrect — don't useMemo for cheap operations:
function Title({ text }: { text: string }) {
  // This is overkill — string operations are fast
  const upperText = useMemo(() => text.toUpperCase(), [text]);
  return <h1>{upperText}</h1>;
}
```

---

### 3. useCallback — Stable Function References

```tsx
import { useCallback, useState } from 'react';

function ProductList({ products }: { products: Product[] }) {
  const [cart, setCart] = useState<string[]>([]);

  // Correct — stable reference prevents child re-renders
  const handleAddToCart = useCallback((sku: string) => {
    setCart(prev => [...prev, sku]);
  }, []);

  // Correct — event handler with dependency
  const handleRemoveFromCart = useCallback((sku: string) => {
    setCart(prev => prev.filter(s => s !== sku));
  }, []);

  return (
    <div className="cmp-product-list">
      {products.map(p => (
        <ProductCard
          key={p.sku}
          product={p}
          onAddToCart={handleAddToCart}
          inCart={cart.includes(p.sku)}
        />
      ))}
    </div>
  );
}

// Incorrect — inline arrow creates new reference every render:
function ProductListBad({ products }: { products: Product[] }) {
  const [cart, setCart] = useState<string[]>([]);

  return (
    <div>
      {products.map(p => (
        <ProductCard
          key={p.sku}
          product={p}
          onAddToCart={(sku) => setCart(prev => [...prev, sku])} // new ref every render!
        />
      ))}
    </div>
  );
}
```

---

### 4. Code Splitting with lazy + Suspense

```tsx
import { lazy, Suspense } from 'react';

// Split by route/feature — not by individual component
const SearchResults = lazy(() => import('@features/search'));
const ProductDetail = lazy(() => import('@features/product-detail'));
const CheckoutFlow = lazy(() => import('@features/checkout'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/search" element={<SearchResults />} />
        <Route path="/products/:slug" element={<ProductDetail />} />
        <Route path="/checkout" element={<CheckoutFlow />} />
      </Routes>
    </Suspense>
  );
}

// Nested Suspense for granular loading:
function ProductPage({ slug }: { slug: string }) {
  return (
    <div>
      {/* Product info loads first */}
      <Suspense fallback={<ProductInfoSkeleton />}>
        <ProductInfo slug={slug} />
      </Suspense>
      {/* Reviews load independently */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews slug={slug} />
      </Suspense>
    </div>
  );
}
```

---

### 5. useTransition for Non-Blocking Updates

Keep the UI responsive during expensive state updates:

```tsx
import { useState, useTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (term: string) => {
    // Urgent: update the input immediately
    setQuery(term);

    // Non-urgent: defer the expensive results update
    startTransition(() => {
      const filtered = filterLargeDataset(term);
      setResults(filtered);
    });
  };

  return (
    <div>
      <input
        value={query}
        onChange={e => handleSearch(e.target.value)}
        className="cmp-search__input"
      />
      {isPending && <div className="cmp-search__loading">Filtering...</div>}
      <ResultList items={results} />
    </div>
  );
}
```

---

### 6. useDeferredValue for Derived Rendering

```tsx
import { useDeferredValue, useMemo } from 'react';

function FilterableList({ items, filter }: { items: Item[]; filter: string }) {
  // Deferred value lags behind the actual filter
  const deferredFilter = useDeferredValue(filter);
  const isStale = filter !== deferredFilter;

  const filteredItems = useMemo(
    () => items.filter(item => item.name.toLowerCase().includes(deferredFilter.toLowerCase())),
    [items, deferredFilter]
  );

  return (
    <div style={{ opacity: isStale ? 0.7 : 1, transition: 'opacity 0.2s' }}>
      <ul>
        {filteredItems.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

---

### 7. Virtualized Lists for Large Datasets

When rendering 100+ items (search results, product grids):

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

function VirtualizedResults({ results }: { results: SearchResult[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: results.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 120, // estimated row height
    overscan: 5,
  });

  return (
    <div
      ref={parentRef}
      className="cmp-results__scroll"
      style={{ height: '600px', overflow: 'auto' }}
    >
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.key}
            className="cmp-results__item"
            style={{
              position: 'absolute',
              top: 0,
              transform: `translateY(${virtualRow.start}px)`,
              height: `${virtualRow.size}px`,
              width: '100%',
            }}
          >
            <ResultCard result={results[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

### 8. Image Optimization for AEM

```tsx
// Leverage AEM Dynamic Media / web-optimized delivery
function OptimizedImage({
  src,
  alt,
  width,
  height,
  priority = false,
}: {
  src: string;
  alt: string;
  width: number;
  height: number;
  priority?: boolean;
}) {
  // AEM web-optimized delivery with responsive widths
  const srcSet = [320, 640, 960, 1280, 1920]
    .map(w => `${src}?width=${w}&quality=85&format=webply ${w}w`)
    .join(', ');

  return (
    <img
      src={`${src}?width=${width}&quality=85&format=webply`}
      srcSet={srcSet}
      sizes={`(max-width: ${width}px) 100vw, ${width}px`}
      alt={alt}
      width={width}
      height={height}
      loading={priority ? 'eager' : 'lazy'}
      decoding={priority ? 'sync' : 'async'}
      fetchPriority={priority ? 'high' : 'auto'}
    />
  );
}
```

---

### 9. Bundle Analysis & Optimization

```bash
# Analyze bundle size
npx webpack-bundle-analyzer stats.json

# In webpack.prod.js — tree-shake unused code
module.exports = {
  optimization: {
    usedExports: true,
    sideEffects: true,
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
        // Separate heavy libraries
        tanstack: {
          test: /[\\/]node_modules[\\/]@tanstack/,
          name: 'tanstack',
          chunks: 'all',
          priority: 10,
        },
      },
    },
  },
};
```

**Import cost awareness:**

```typescript
// Correct — named import, tree-shakeable:
import { useQuery } from '@tanstack/react-query';

// Incorrect — imports entire library:
import * as ReactQuery from '@tanstack/react-query';

// Correct — dynamic import for optional features:
const loadChartLib = () => import('chart.js/auto');
```

---

### 10. Core Web Vitals Alignment

| Metric | Target | React Pattern |
|--------|--------|--------------|
| LCP < 2.5s | Prioritize above-fold images, SSR/SSG | `fetchPriority="high"`, preload, `loading="eager"` for hero |
| INP < 200ms | Keep interactions responsive | `useTransition`, `useDeferredValue`, debounce |
| CLS < 0.1 | Prevent layout shift | Set explicit `width`/`height` on images, skeleton loaders |

```tsx
// Skeleton loader to prevent CLS
function CardSkeleton() {
  return (
    <div className="cmp-card cmp-card--skeleton" aria-hidden="true">
      <div className="cmp-card__image-placeholder" style={{ aspectRatio: '16/9' }} />
      <div className="cmp-card__title-placeholder" style={{ height: '24px', width: '80%' }} />
      <div className="cmp-card__desc-placeholder" style={{ height: '16px', width: '60%' }} />
    </div>
  );
}
```

---

### 11. Optimizing Interactions with requestAnimationFrame and setTimeout

Use `requestAnimationFrame` (rAF) for visual updates and `setTimeout(fn, 0)` to yield back to the browser during heavy work:

```tsx
import { useRef, useCallback } from 'react';

// rAF for smooth visual updates (scroll, resize, drag)
function useAnimatedValue() {
  const rafId = useRef<number>(0);

  const update = useCallback((el: HTMLElement, value: number) => {
    cancelAnimationFrame(rafId.current);
    rafId.current = requestAnimationFrame(() => {
      el.style.transform = `translateX(${value}px)`;
    });
  }, []);

  return update;
}

// Debounce with rAF — at most one update per frame (16ms)
function useRafCallback<T extends (...args: any[]) => void>(callback: T): T {
  const rafId = useRef<number>(0);
  const latestArgs = useRef<Parameters<T>>();

  return useCallback((...args: Parameters<T>) => {
    latestArgs.current = args;
    cancelAnimationFrame(rafId.current);
    rafId.current = requestAnimationFrame(() => {
      callback(...latestArgs.current!);
    });
  }, [callback]) as T;
}

// Usage — smooth scroll tracking:
function ParallaxHero({ image }: { image: string }) {
  const heroRef = useRef<HTMLDivElement>(null);

  const handleScroll = useRafCallback(() => {
    if (!heroRef.current) return;
    const scroll = window.scrollY;
    heroRef.current.style.transform = `translateY(${scroll * 0.3}px)`;
  });

  useEffect(() => {
    window.addEventListener('scroll', handleScroll, { passive: true });
    return () => window.removeEventListener('scroll', handleScroll);
  }, [handleScroll]);

  return <div ref={heroRef} className="cmp-hero__bg" />;
}
```

**setTimeout(0) to yield to the browser — keep INP low:**

```tsx
// Break up heavy work so the browser can paint between chunks
async function processLargeList(items: Item[], batchSize = 50) {
  const results: ProcessedItem[] = [];

  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    results.push(...batch.map(processItem));

    // Yield to browser — allows paint, input handling
    if (i + batchSize < items.length) {
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }

  return results;
}

// In a component:
function HeavyFilterList({ items, filter }: { items: Item[]; filter: string }) {
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  useEffect(() => {
    // Combine useTransition with yielding for truly heavy work
    startTransition(() => {
      processLargeList(items.filter(i => i.name.includes(filter)))
        .then(setResults);
    });
  }, [items, filter]);

  return (
    <div style={{ opacity: isPending ? 0.7 : 1 }}>
      {results.map(r => <ItemRow key={r.id} item={r} />)}
    </div>
  );
}
```

**When to use which:**

| Technique | Use For | Timing |
|-----------|---------|--------|
| `requestAnimationFrame` | Visual updates (transforms, opacity, scroll) | Before next paint (~16ms) |
| `setTimeout(fn, 0)` | Yield to browser during heavy JS work | After current task, before next task |
| `useTransition` | React state updates that can be deferred | React scheduler manages priority |
| `useDeferredValue` | Derived values that can lag behind | React scheduler manages priority |
| `scheduler.yield()` | Modern yield API (Chrome 129+) | Explicit yield point in long tasks |

---

### Anti-Patterns

- **Memoizing everything** — `memo`, `useMemo`, `useCallback` have overhead; only use when re-renders are measurably expensive
- **useMemo for cheap operations** — string concatenation, simple math don't need memoization
- **useCallback without memo on child** — `useCallback` is pointless if the child isn't wrapped in `memo`
- **Inline objects/arrays as props** — `style={{ color: 'red' }}` creates a new object every render; extract to constant or useMemo
- **Not using keys properly** — array index as key causes unnecessary re-mounts; use stable IDs
- **Loading all features eagerly** — use `lazy()` for below-fold features and non-critical routes
- **Ignoring bundle size** — audit with webpack-bundle-analyzer; set bundle budgets in CI
- **Images without dimensions** — causes CLS; always set `width` and `height` attributes
- **Blocking renders with synchronous work** — use `useTransition` for expensive state updates
- **Not profiling before optimizing** — use React DevTools Profiler to find actual bottlenecks, not guessed ones
