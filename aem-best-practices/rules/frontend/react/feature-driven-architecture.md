---
title: React Feature-Driven Architecture for AEM
impact: HIGH
impactDescription: Proper folder structure prevents spaghetti code and enables team scalability in AEM SPA projects
tags: react, architecture, feature-driven, domain, folder-structure, lazy-loading, barrel-exports
---

## React Feature-Driven Architecture for AEM

Organize AEM React SPAs by domain feature, not by file type. Each feature is a self-contained module with its own components, hooks, context, styles, and tests — making code ownership clear and enabling lazy loading per feature.

---

### 1. Feature Folder Structure

**Correct — feature-driven layout:**

```
ui.frontend/src/
├── features/
│   ├── hero/
│   │   ├── index.ts                  # barrel export
│   │   ├── Hero.tsx                   # main component (MapTo target)
│   │   ├── HeroImage.tsx              # sub-component
│   │   ├── HeroContent.tsx            # sub-component
│   │   ├── useHeroAnimation.ts        # feature-specific hook
│   │   ├── hero.context.tsx           # feature-local context (if needed)
│   │   ├── hero.types.ts              # TypeScript interfaces
│   │   ├── hero.module.scss           # scoped styles
│   │   └── __tests__/
│   │       ├── Hero.test.tsx
│   │       └── useHeroAnimation.test.ts
│   ├── navigation/
│   │   ├── index.ts
│   │   ├── MegaMenu.tsx
│   │   ├── MegaMenuItem.tsx
│   │   ├── useNavigation.ts
│   │   ├── navigation.context.tsx
│   │   ├── navigation.types.ts
│   │   ├── navigation.module.scss
│   │   └── __tests__/
│   ├── search/
│   │   ├── index.ts
│   │   ├── SearchBar.tsx
│   │   ├── SearchResults.tsx
│   │   ├── SearchFacets.tsx
│   │   ├── useSearch.ts
│   │   ├── search.context.tsx
│   │   ├── search.types.ts
│   │   └── __tests__/
│   └── product-detail/
│       ├── index.ts
│       ├── ProductDetail.tsx
│       ├── ProductGallery.tsx
│       ├── ProductSpecs.tsx
│       ├── useProduct.ts
│       ├── product.types.ts
│       └── __tests__/
├── shared/
│   ├── components/                    # truly shared UI primitives
│   │   ├── Button/
│   │   ├── Modal/
│   │   └── Spinner/
│   ├── hooks/                         # cross-feature hooks
│   │   ├── useBreakpoint.ts
│   │   ├── useIntersection.ts
│   │   └── useAuthorMode.ts
│   ├── context/                       # app-wide providers
│   │   ├── AemPageContext.tsx
│   │   └── ThemeContext.tsx
│   ├── utils/                         # pure utility functions
│   │   ├── aem.ts
│   │   └── format.ts
│   └── types/                         # shared TypeScript types
│       └── aem.types.ts
├── App.tsx
├── index.tsx
└── import-components.ts               # MapTo registrations
```

**Incorrect — file-type-driven layout (don't do this):**

```
ui.frontend/src/
├── components/          # 50+ components dumped here
│   ├── Hero.tsx
│   ├── HeroImage.tsx
│   ├── MegaMenu.tsx
│   ├── SearchBar.tsx
│   └── ...
├── hooks/               # all hooks mixed together
│   ├── useHeroAnimation.ts
│   ├── useNavigation.ts
│   └── useSearch.ts
├── context/             # all contexts mixed
├── styles/              # all styles mixed
└── tests/               # all tests mixed
```

---

### 2. Barrel Exports

Each feature exposes a clean public API through `index.ts`:

```typescript
// features/hero/index.ts
export { Hero } from './Hero';
export { HeroImage } from './HeroImage';
export type { HeroProps, HeroModel } from './hero.types';

// features/search/index.ts
export { SearchBar } from './SearchBar';
export { SearchResults } from './SearchResults';
export type { SearchProps, SearchQuery } from './search.types';
```

**Importing from other features:**

```typescript
// Correct — import from barrel
import { Hero } from '@features/hero';
import { SearchBar } from '@features/search';

// Incorrect — reaching into feature internals
import { Hero } from '@features/hero/Hero';
import { useHeroAnimation } from '@features/hero/useHeroAnimation';
```

**Webpack alias in `ui.frontend/webpack.common.js`:**

```javascript
resolve: {
  alias: {
    '@features': path.resolve(__dirname, 'src/features'),
    '@shared': path.resolve(__dirname, 'src/shared'),
  },
}
```

---

### 3. Lazy-Loaded Feature Modules

Split large features into separate chunks:

```typescript
// import-components.ts
import { MapTo } from '@adobe/aem-react-editable-components';
import { lazy } from 'react';

// Eager — above-the-fold, critical path
import { Hero } from '@features/hero';
MapTo('mysite/components/hero')(Hero);

// Lazy — below-the-fold or conditional
const SearchResults = lazy(() =>
  import('@features/search').then(m => ({ default: m.SearchResults }))
);
MapTo('mysite/components/search-results')(SearchResults);

const ProductDetail = lazy(() =>
  import('@features/product-detail').then(m => ({ default: m.ProductDetail }))
);
MapTo('mysite/components/product-detail')(ProductDetail);
```

**Suspense boundary in the page layout:**

```tsx
import { Suspense } from 'react';
import { Spinner } from '@shared/components/Spinner';

export const PageContent: React.FC<{ cqItems: any }> = ({ cqItems }) => (
  <Suspense fallback={<Spinner />}>
    <ResponsiveGrid {...cqItems} />
  </Suspense>
);
```

---

### 4. Feature-Specific Context

Keep context scoped to the feature that needs it:

```tsx
// features/search/search.context.tsx
import { createContext, useContext, useReducer, type ReactNode } from 'react';
import type { SearchState, SearchAction } from './search.types';

const SearchContext = createContext<SearchState | null>(null);
const SearchDispatchContext = createContext<React.Dispatch<SearchAction> | null>(null);

function searchReducer(state: SearchState, action: SearchAction): SearchState {
  switch (action.type) {
    case 'SET_QUERY':
      return { ...state, query: action.payload, page: 1 };
    case 'SET_RESULTS':
      return { ...state, results: action.payload, loading: false };
    case 'SET_FACET':
      return { ...state, facets: { ...state.facets, [action.key]: action.value }, page: 1 };
    case 'SET_PAGE':
      return { ...state, page: action.payload };
    default:
      return state;
  }
}

export function SearchProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(searchReducer, {
    query: '',
    results: [],
    facets: {},
    page: 1,
    loading: false,
  });

  return (
    <SearchContext.Provider value={state}>
      <SearchDispatchContext.Provider value={dispatch}>
        {children}
      </SearchDispatchContext.Provider>
    </SearchContext.Provider>
  );
}

export function useSearchState() {
  const ctx = useContext(SearchContext);
  if (!ctx) throw new Error('useSearchState must be within SearchProvider');
  return ctx;
}

export function useSearchDispatch() {
  const ctx = useContext(SearchDispatchContext);
  if (!ctx) throw new Error('useSearchDispatch must be within SearchProvider');
  return ctx;
}
```

---

### 5. Shared vs Feature-Specific Decision Rules

| Criterion | Shared (`shared/`) | Feature (`features/X/`) |
|-----------|-------------------|------------------------|
| Used by 3+ features | Yes | No |
| Has AEM-specific logic | `shared/hooks/` | Feature hook |
| Pure UI primitive (Button, Modal) | Yes | No |
| Contains business logic | No | Yes |
| Feature-specific state | No | Yes |
| Re-exported by multiple barrels | Promote to shared | Keep in feature |

**Rule of thumb**: Start in the feature folder. Only move to `shared/` when a third feature needs it.

---

### 6. MapTo Registration Pattern

Centralize AEM component registration in a single file:

```typescript
// import-components.ts — the AEM ↔ React bridge
import { MapTo, withMappable } from '@adobe/aem-react-editable-components';

// Feature imports (from barrels)
import { Hero } from '@features/hero';
import { MegaMenu } from '@features/navigation';
import { SearchBar, SearchResults } from '@features/search';
import { ProductDetail } from '@features/product-detail';

// Edit configs
const HeroEditConfig = {
  emptyLabel: 'Hero Banner',
  isEmpty: (props: any) => !props.title && !props.backgroundImage,
};

// Registration
MapTo('mysite/components/hero')(Hero, HeroEditConfig);
MapTo('mysite/components/mega-menu')(MegaMenu);
MapTo('mysite/components/search-bar')(SearchBar);
MapTo('mysite/components/search-results')(SearchResults);
MapTo('mysite/components/product-detail')(ProductDetail);
```

---

### 7. TypeScript Types per Feature

```typescript
// features/hero/hero.types.ts
import type { MappedComponentProperties } from '@adobe/aem-react-editable-components';

export interface HeroModel {
  title: string;
  subtitle?: string;
  backgroundImage: string;
  ctaLabel?: string;
  ctaLink?: string;
  overlayOpacity?: number;
}

export interface HeroProps extends HeroModel, MappedComponentProperties {}

// features/search/search.types.ts
export interface SearchQuery {
  term: string;
  facets: Record<string, string[]>;
  page: number;
  pageSize: number;
}

export interface SearchResult {
  path: string;
  title: string;
  description: string;
  thumbnail: string;
  score: number;
}

export interface SearchState {
  query: string;
  results: SearchResult[];
  facets: Record<string, string[]>;
  page: number;
  loading: boolean;
}

export type SearchAction =
  | { type: 'SET_QUERY'; payload: string }
  | { type: 'SET_RESULTS'; payload: SearchResult[] }
  | { type: 'SET_FACET'; key: string; value: string[] }
  | { type: 'SET_PAGE'; payload: number };
```

---

### Anti-Patterns

- **Flat `components/` dump** — 50+ components in one folder with no grouping
- **Cross-feature imports** — reaching into `features/search/useSearch.ts` from `features/hero/`
- **God context** — single AppContext holding all state for all features
- **Hooks folder soup** — all hooks in `hooks/` with no ownership clarity
- **Premature sharing** — moving a component to `shared/` after only 1 feature uses it
- **Missing barrel exports** — importing deep paths instead of feature API
- **Eager loading everything** — no code splitting for below-the-fold features
- **Types scattered** — interfaces defined inline in components instead of `.types.ts`
