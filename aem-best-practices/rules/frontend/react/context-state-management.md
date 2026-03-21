---
title: React Context & State Management for AEM
impact: HIGH
impactDescription: Poor state management in AEM SPAs causes unnecessary re-renders, broken author mode, and hydration mismatches
tags: react, context, state-management, useReducer, provider, aem-spa, prop-drilling
---

## React Context & State Management for AEM

AEM React SPAs need state management that works across publish and author modes, respects the AEM page model lifecycle, and avoids prop drilling through deeply nested component trees.

---

### 1. AEM Page Context Provider

Wrap the entire SPA with a context that exposes AEM page model data:

```tsx
// shared/context/AemPageContext.tsx
import { createContext, useContext, useEffect, useState, type ReactNode } from 'react';
import { ModelManager } from '@adobe/aem-spa-page-model-manager';

interface AemPageData {
  pagePath: string;
  pageTitle: string;
  locale: string;
  isAuthorMode: boolean;
  wcmMode: 'EDIT' | 'PREVIEW' | 'DISABLED';
  dataLayer: Record<string, unknown>;
}

const AemPageContext = createContext<AemPageData | null>(null);

export function AemPageProvider({ children }: { children: ReactNode }) {
  const [pageData, setPageData] = useState<AemPageData>(() => ({
    pagePath: '',
    pageTitle: '',
    locale: 'en',
    isAuthorMode: false,
    wcmMode: 'DISABLED',
    dataLayer: {},
  }));

  useEffect(() => {
    const meta = document.querySelector('meta[name="cq:wcmmode"]');
    const wcmMode = (meta?.getAttribute('content') as AemPageData['wcmMode']) || 'DISABLED';

    ModelManager.getData().then(model => {
      setPageData({
        pagePath: model[':path'] || '',
        pageTitle: model['jcr:title'] || '',
        locale: model.language || 'en',
        isAuthorMode: wcmMode !== 'DISABLED',
        wcmMode,
        dataLayer: model.dataLayer || {},
      });
    });
  }, []);

  return (
    <AemPageContext.Provider value={pageData}>
      {children}
    </AemPageContext.Provider>
  );
}

export function useAemPage(): AemPageData {
  const ctx = useContext(AemPageContext);
  if (!ctx) throw new Error('useAemPage must be within AemPageProvider');
  return ctx;
}
```

---

### 2. Compound Provider Pattern

Compose multiple providers without nesting hell:

```tsx
// shared/context/AppProviders.tsx
import type { ReactNode } from 'react';
import { AemPageProvider } from './AemPageContext';
import { ThemeProvider } from './ThemeContext';
import { AuthProvider } from '@features/auth';

interface ProvidersProps {
  children: ReactNode;
}

// Utility to compose providers left-to-right
function composeProviders(...providers: React.FC<{ children: ReactNode }>[]) {
  return ({ children }: ProvidersProps) =>
    providers.reduceRight(
      (acc, Provider) => <Provider>{acc}</Provider>,
      children
    );
}

export const AppProviders = composeProviders(
  AemPageProvider,
  ThemeProvider,
  AuthProvider
);

// Usage in App.tsx
export default function App() {
  return (
    <AppProviders>
      <Page />
    </AppProviders>
  );
}
```

**Incorrect — deeply nested providers:**

```tsx
// Don't do this
export default function App() {
  return (
    <AemPageProvider>
      <ThemeProvider>
        <AuthProvider>
          <SearchProvider>
            <CartProvider>
              <Page />
            </CartProvider>
          </SearchProvider>
        </AuthProvider>
      </ThemeProvider>
    </AemPageProvider>
  );
}
```

---

### 3. useReducer for Complex Feature State

```tsx
// features/cart/cart.context.tsx
import { createContext, useContext, useReducer, type ReactNode, type Dispatch } from 'react';

interface CartItem {
  sku: string;
  title: string;
  price: number;
  quantity: number;
}

interface CartState {
  items: CartItem[];
  total: number;
  currency: string;
  loading: boolean;
}

type CartAction =
  | { type: 'ADD_ITEM'; payload: CartItem }
  | { type: 'REMOVE_ITEM'; sku: string }
  | { type: 'UPDATE_QUANTITY'; sku: string; quantity: number }
  | { type: 'CLEAR' }
  | { type: 'SET_LOADING'; payload: boolean };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existing = state.items.find(i => i.sku === action.payload.sku);
      const items = existing
        ? state.items.map(i =>
            i.sku === action.payload.sku
              ? { ...i, quantity: i.quantity + action.payload.quantity }
              : i
          )
        : [...state.items, action.payload];
      return { ...state, items, total: items.reduce((s, i) => s + i.price * i.quantity, 0) };
    }
    case 'REMOVE_ITEM': {
      const items = state.items.filter(i => i.sku !== action.sku);
      return { ...state, items, total: items.reduce((s, i) => s + i.price * i.quantity, 0) };
    }
    case 'UPDATE_QUANTITY': {
      const items = state.items.map(i =>
        i.sku === action.sku ? { ...i, quantity: action.quantity } : i
      );
      return { ...state, items, total: items.reduce((s, i) => s + i.price * i.quantity, 0) };
    }
    case 'CLEAR':
      return { ...state, items: [], total: 0 };
    case 'SET_LOADING':
      return { ...state, loading: action.payload };
    default:
      return state;
  }
}

const CartStateCtx = createContext<CartState | null>(null);
const CartDispatchCtx = createContext<Dispatch<CartAction> | null>(null);

export function CartProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, {
    items: [],
    total: 0,
    currency: 'USD',
    loading: false,
  });

  return (
    <CartStateCtx.Provider value={state}>
      <CartDispatchCtx.Provider value={dispatch}>
        {children}
      </CartDispatchCtx.Provider>
    </CartStateCtx.Provider>
  );
}

export const useCartState = () => {
  const ctx = useContext(CartStateCtx);
  if (!ctx) throw new Error('useCartState must be within CartProvider');
  return ctx;
};

export const useCartDispatch = () => {
  const ctx = useContext(CartDispatchCtx);
  if (!ctx) throw new Error('useCartDispatch must be within CartProvider');
  return ctx;
};
```

---

### 4. Persisted State with Storage Sync

Context state that survives page reloads:

```tsx
// shared/hooks/usePersistedReducer.ts
import { useReducer, useEffect, useRef, type Reducer } from 'react';

export function usePersistedReducer<S, A>(
  reducer: Reducer<S, A>,
  initialState: S,
  storageKey: string,
  storage: Storage = sessionStorage
): [S, React.Dispatch<A>] {
  const [state, dispatch] = useReducer(reducer, initialState, (init) => {
    try {
      const stored = storage.getItem(storageKey);
      return stored ? JSON.parse(stored) : init;
    } catch {
      return init;
    }
  });

  const stateRef = useRef(state);
  stateRef.current = state;

  useEffect(() => {
    try {
      storage.setItem(storageKey, JSON.stringify(stateRef.current));
    } catch {
      // quota exceeded — degrade gracefully
    }
  }, [state, storageKey, storage]);

  return [state, dispatch];
}

// Usage in CartProvider:
const [state, dispatch] = usePersistedReducer(
  cartReducer,
  { items: [], total: 0, currency: 'USD', loading: false },
  'mysite:cart',
  localStorage
);
```

---

### 5. Global vs Local State Boundaries

| State Type | Scope | Pattern | Example |
|-----------|-------|---------|---------|
| AEM page model | Global | `AemPageProvider` | Page path, locale, WCM mode |
| Theme / locale | Global | `ThemeProvider` | Dark mode, language |
| Auth / user | Global | `AuthProvider` | Login state, tokens |
| Feature data | Feature | Feature context | Search results, cart items |
| Component UI | Component | `useState` | Dropdown open, input value |
| Server data | Feature | TanStack Query / SWR | CF data, GraphQL results |

**Rule**: If state is only read by components within one feature, keep it in that feature's context. Promote to global only when 3+ features need it.

---

### 6. Author Mode Considerations

AEM author mode re-renders components on drag-drop and dialog save. Handle this:

```tsx
// shared/hooks/useAuthorMode.ts
import { useAemPage } from '@shared/context/AemPageContext';
import { useEffect, useCallback } from 'react';

export function useAuthorMode() {
  const { isAuthorMode, wcmMode } = useAemPage();

  const onDialogSave = useCallback((callback: () => void) => {
    if (!isAuthorMode) return;
    // Granite author channel events
    const channel = new BroadcastChannel('aem-author');
    const handler = (e: MessageEvent) => {
      if (e.data?.type === 'dialog-save') callback();
    };
    channel.addEventListener('message', handler);
    return () => channel.close();
  }, [isAuthorMode]);

  return { isAuthorMode, wcmMode, onDialogSave };
}

// In a component:
function Hero({ title, backgroundImage }: HeroProps) {
  const { isAuthorMode } = useAuthorMode();

  // Don't lazy-load in author mode — author needs instant preview
  if (isAuthorMode) {
    return <HeroContent title={title} backgroundImage={backgroundImage} />;
  }

  return (
    <IntersectionWrapper>
      <HeroContent title={title} backgroundImage={backgroundImage} />
    </IntersectionWrapper>
  );
}
```

---

### Anti-Patterns

- **Single global store for everything** — one giant context re-renders the entire app on any change
- **Not splitting state and dispatch contexts** — causes all consumers to re-render on dispatch even if they only read state
- **Using Context for server data** — use TanStack Query/SWR for data fetching, Context for client state
- **Ignoring author mode** — lazy loading and intersection observers break drag-drop in the page editor
- **Mutable refs for shared state** — refs don't trigger re-renders, causing stale UI
- **Direct localStorage in render** — causes hydration mismatches in SSR; use lazy initializer in useReducer
- **Provider outside the feature** — wrapping `SearchProvider` at the app root when only search components need it
