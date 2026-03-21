---
title: Preact Signals for AEM State Management
impact: HIGH
impactDescription: Signals provide fine-grained reactivity without re-rendering entire component trees — critical for performance in AEM interactive islands
tags: preact, signals, state-management, reactive, computed, effect, performance
---

## Preact Signals for AEM State Management

Preact Signals offer fine-grained reactivity — only the DOM nodes that read a signal update when it changes, bypassing the virtual DOM diffing entirely. This makes them ideal for AEM interactive islands where performance matters.

---

### 1. Signal Basics

```typescript
import { signal, computed, effect } from '@preact/signals';

// Primitive signal
const count = signal(0);
count.value++;        // triggers updates in reading components
console.log(count.value); // 1

// Computed signal (derived state)
const doubled = computed(() => count.value * 2);
console.log(doubled.value); // 2

// Effect (side effects)
const dispose = effect(() => {
  console.log('Count changed:', count.value);
});
// Call dispose() to clean up
```

---

### 2. Signal Store Pattern for AEM

```typescript
// features/search/search.store.ts
import { signal, computed, effect } from '@preact/signals';

// State signals
export const query = signal('');
export const results = signal<SearchResult[]>([]);
export const loading = signal(false);
export const selectedFacets = signal<Record<string, string[]>>({});
export const currentPage = signal(1);

// Computed
export const resultCount = computed(() => results.value.length);
export const hasResults = computed(() => results.value.length > 0);
export const activeFilters = computed(() =>
  Object.entries(selectedFacets.value)
    .filter(([, values]) => values.length > 0)
    .map(([key, values]) => ({ key, values }))
);

// Actions (plain functions that mutate signals)
export async function search(term: string) {
  query.value = term;
  currentPage.value = 1;
  loading.value = true;

  try {
    const response = await fetch(
      `/bin/mysite/search?q=${encodeURIComponent(term)}&page=1`
    );
    const data = await response.json();
    results.value = data.results;
  } catch (err) {
    console.error('Search failed:', err);
    results.value = [];
  } finally {
    loading.value = false;
  }
}

export function toggleFacet(key: string, value: string) {
  const current = selectedFacets.value[key] || [];
  const updated = current.includes(value)
    ? current.filter(v => v !== value)
    : [...current, value];

  selectedFacets.value = {
    ...selectedFacets.value,
    [key]: updated,
  };
}

export function clearFilters() {
  selectedFacets.value = {};
  currentPage.value = 1;
}

// Persist query to URL
effect(() => {
  const url = new URL(window.location.href);
  if (query.value) {
    url.searchParams.set('q', query.value);
  } else {
    url.searchParams.delete('q');
  }
  window.history.replaceState(null, '', url.toString());
});
```

---

### 3. Using Signals in Components

```tsx
// features/search/SearchBar.tsx
import { query, search, loading } from './search.store';

export function SearchBar() {
  const handleSubmit = (e: Event) => {
    e.preventDefault();
    search(query.value);
  };

  return (
    <form class="cmp-search__form" onSubmit={handleSubmit}>
      <input
        class="cmp-search__input"
        type="search"
        value={query}  {/* Signal directly — auto-subscribes */}
        onInput={(e) => { query.value = (e.target as HTMLInputElement).value; }}
        placeholder="Search..."
      />
      <button class="cmp-search__button" type="submit" disabled={loading}>
        {loading.value ? 'Searching...' : 'Search'}
      </button>
    </form>
  );
}

// features/search/SearchResults.tsx
import { results, hasResults, loading, resultCount } from './search.store';

export function SearchResults() {
  if (loading.value) return <div class="cmp-search__loading">Loading...</div>;
  if (!hasResults.value) return <div class="cmp-search__empty">No results</div>;

  return (
    <div class="cmp-search__results">
      <p class="cmp-search__count">{resultCount} results</p>
      <ul class="cmp-search__list">
        {results.value.map(result => (
          <li key={result.path} class="cmp-search__item">
            <a href={result.path}>{result.title}</a>
            <p>{result.description}</p>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

**Key**: When you pass a signal directly as a JSX attribute (`value={query}` not `value={query.value}`), Preact subscribes at the DOM level — the component function never re-executes, only the text node updates.

---

### 4. Signals vs Context Performance

```tsx
// Context approach — entire tree re-renders on any state change:
function SearchProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  // Every child re-renders when ANY state property changes
  return <SearchContext.Provider value={{ state, dispatch }}>{children}</SearchContext.Provider>;
}

// Signals approach — only reading components update:
// SearchBar re-renders only when `query` changes
// SearchResults re-renders only when `results` changes
// SearchFacets re-renders only when `selectedFacets` changes
// No provider wrapper needed
```

---

### 5. Signal-Based AEM Content Sync

Sync AEM component data with signals:

```typescript
// shared/signals/aem-content.ts
import { signal, effect } from '@preact/signals';

export function createContentSignal<T>(
  initialData: T,
  refreshUrl?: string
) {
  const data = signal<T>(initialData);
  const stale = signal(false);

  // Listen for AEM author save events
  if (typeof window !== 'undefined' && window.CQ?.WCM) {
    document.addEventListener('cq-editables-updated', () => {
      stale.value = true;
      if (refreshUrl) {
        fetch(refreshUrl)
          .then(r => r.json())
          .then(json => {
            data.value = json as T;
            stale.value = false;
          });
      }
    });
  }

  return { data, stale };
}

// Usage:
const { data: heroData } = createContentSignal(
  { title: 'Initial', image: '' },
  '/content/mysite/en/jcr:content/root/hero.model.json'
);
```

---

### 6. DevTools & Debugging

```typescript
// Enable signal tracking in development
import { signal } from '@preact/signals';

if (process.env.NODE_ENV === 'development') {
  // Log all signal changes
  const originalSignal = signal;
  (window as any).__PREACT_SIGNALS__ = new Map();

  // Use @preact/signals-react-devtools or console tracking
  effect(() => {
    console.group('Signal State Snapshot');
    console.log('query:', query.value);
    console.log('results:', results.value.length);
    console.log('loading:', loading.value);
    console.groupEnd();
  });
}
```

---

### Anti-Patterns

- **Using `.value` in JSX when signal can be passed directly** — `<span>{count}</span>` is more efficient than `<span>{count.value}</span>` (the former updates only the text node)
- **Creating signals inside components** — signals should be created outside components (module scope) or in stores; creating inside causes new signals on every render
- **Mutating signal arrays/objects in place** — `results.value.push(item)` won't trigger updates; use `results.value = [...results.value, item]`
- **Using signals for everything** — local UI state (`useState`) is fine for component-internal toggle, hover, focus
- **Not disposing effects** — effects created in component lifecycle must be disposed on unmount
- **Mixing signals and Context in the same feature** — pick one reactive model per feature
