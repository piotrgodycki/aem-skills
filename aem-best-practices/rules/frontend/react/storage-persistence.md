---
title: Storage & Persistence Patterns for AEM React SPAs
impact: MEDIUM
impactDescription: Incorrect storage handling causes hydration mismatches, lost user state, and cross-site data leaks in multi-tenant AEM
tags: react, storage, localStorage, sessionStorage, indexeddb, cookies, persistence, hydration
---

## Storage & Persistence Patterns for AEM React SPAs

Storage in AEM React applications must handle multi-tenant namespace isolation, SSR/hydration safety, cross-tab synchronization, and graceful quota fallbacks.

---

### 1. Typed Storage Wrapper

```typescript
// shared/utils/storage.ts
interface StorageOptions {
  namespace?: string;
  ttl?: number;         // milliseconds
  storage?: Storage;
}

interface StorageEntry<T> {
  value: T;
  expires?: number;
}

export function createStorage(defaults: StorageOptions = {}) {
  const {
    namespace = 'aem',
    ttl,
    storage = localStorage,
  } = defaults;

  function key(k: string): string {
    return `${namespace}:${k}`;
  }

  return {
    get<T>(k: string, fallback: T): T {
      try {
        const raw = storage.getItem(key(k));
        if (!raw) return fallback;
        const entry: StorageEntry<T> = JSON.parse(raw);
        if (entry.expires && Date.now() > entry.expires) {
          storage.removeItem(key(k));
          return fallback;
        }
        return entry.value;
      } catch {
        return fallback;
      }
    },

    set<T>(k: string, value: T, itemTtl?: number): void {
      const effectiveTtl = itemTtl ?? ttl;
      const entry: StorageEntry<T> = {
        value,
        ...(effectiveTtl ? { expires: Date.now() + effectiveTtl } : {}),
      };
      try {
        storage.setItem(key(k), JSON.stringify(entry));
      } catch {
        // quota exceeded — evict oldest namespace entries
        this.evictOldest(3);
        try {
          storage.setItem(key(k), JSON.stringify(entry));
        } catch {
          // still full — degrade silently
        }
      }
    },

    remove(k: string): void {
      storage.removeItem(key(k));
    },

    clear(): void {
      const prefix = `${namespace}:`;
      const keysToRemove: string[] = [];
      for (let i = 0; i < storage.length; i++) {
        const k = storage.key(i);
        if (k?.startsWith(prefix)) keysToRemove.push(k);
      }
      keysToRemove.forEach(k => storage.removeItem(k));
    },

    evictOldest(count: number): void {
      const prefix = `${namespace}:`;
      const entries: { key: string; expires: number }[] = [];
      for (let i = 0; i < storage.length; i++) {
        const k = storage.key(i);
        if (!k?.startsWith(prefix)) continue;
        try {
          const entry = JSON.parse(storage.getItem(k)!);
          entries.push({ key: k, expires: entry.expires || 0 });
        } catch {
          entries.push({ key: k, expires: 0 });
        }
      }
      entries
        .sort((a, b) => a.expires - b.expires)
        .slice(0, count)
        .forEach(e => storage.removeItem(e.key));
    },
  };
}

// Per-site namespace for multi-tenant AEM
export const siteStorage = createStorage({ namespace: 'mysite' });
export const sessionStore = createStorage({ namespace: 'mysite', storage: sessionStorage });
```

---

### 2. Hydration-Safe useStorage Hook

Prevents SSR/hydration mismatches by deferring to client:

```typescript
// shared/hooks/useStorage.ts
import { useState, useEffect, useCallback } from 'react';
import { siteStorage } from '@shared/utils/storage';

export function useStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T | ((prev: T) => T)) => void] {
  // Always start with initialValue to match SSR
  const [value, setValue] = useState<T>(initialValue);

  // Hydrate from storage on mount (client only)
  useEffect(() => {
    const stored = siteStorage.get<T>(key, initialValue);
    setValue(stored);
  }, [key]);

  const setStoredValue = useCallback(
    (newValue: T | ((prev: T) => T)) => {
      setValue(prev => {
        const resolved = typeof newValue === 'function'
          ? (newValue as (prev: T) => T)(prev)
          : newValue;
        siteStorage.set(key, resolved);
        return resolved;
      });
    },
    [key]
  );

  return [value, setStoredValue];
}

// Usage:
function ThemeToggle() {
  const [theme, setTheme] = useStorage('theme', 'light');

  return (
    <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
      {theme === 'light' ? '🌙' : '☀️'}
    </button>
  );
}
```

---

### 3. Cross-Tab Synchronization

Keep state consistent across browser tabs:

```typescript
// shared/hooks/useCrossTabSync.ts
import { useEffect, useCallback } from 'react';

export function useCrossTabSync(
  key: string,
  onUpdate: (newValue: string | null) => void
): void {
  useEffect(() => {
    function handleStorage(e: StorageEvent) {
      if (e.key === key) {
        onUpdate(e.newValue);
      }
    }
    window.addEventListener('storage', handleStorage);
    return () => window.removeEventListener('storage', handleStorage);
  }, [key, onUpdate]);
}

// Usage — sync cart across tabs:
function CartProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, initialCart);

  useCrossTabSync('mysite:cart', useCallback((raw) => {
    if (raw) {
      try {
        const synced = JSON.parse(raw);
        dispatch({ type: 'SYNC', payload: synced.value });
      } catch { /* ignore malformed */ }
    }
  }, []));

  return (
    <CartContext.Provider value={{ state, dispatch }}>
      {children}
    </CartContext.Provider>
  );
}
```

---

### 4. URL State Synchronization

For search, filters, and pagination — state lives in the URL:

```typescript
// shared/hooks/useUrlState.ts
import { useCallback, useMemo } from 'react';
import { useSearchParams } from 'react-router-dom';

export function useUrlState<T extends Record<string, string>>(
  defaults: T
): [T, (updates: Partial<T>) => void] {
  const [searchParams, setSearchParams] = useSearchParams();

  const state = useMemo(() => {
    const result = { ...defaults };
    for (const key of Object.keys(defaults)) {
      const param = searchParams.get(key);
      if (param !== null) {
        (result as Record<string, string>)[key] = param;
      }
    }
    return result;
  }, [searchParams, defaults]);

  const setState = useCallback(
    (updates: Partial<T>) => {
      setSearchParams(prev => {
        const next = new URLSearchParams(prev);
        for (const [key, value] of Object.entries(updates)) {
          if (value === defaults[key] || value === '') {
            next.delete(key);
          } else {
            next.set(key, value as string);
          }
        }
        return next;
      });
    },
    [setSearchParams, defaults]
  );

  return [state, setState];
}

// Usage:
function SearchPage() {
  const [params, setParams] = useUrlState({
    q: '',
    category: '',
    page: '1',
  });

  return (
    <>
      <input value={params.q} onChange={e => setParams({ q: e.target.value, page: '1' })} />
      <Pagination page={Number(params.page)} onChange={p => setParams({ page: String(p) })} />
    </>
  );
}
```

---

### 5. IndexedDB for Large Data

When localStorage quota is insufficient (images, offline data):

```typescript
// shared/utils/idb.ts
const DB_NAME = 'aem-mysite';
const DB_VERSION = 1;
const STORE_NAME = 'cache';

function openDB(): Promise<IDBDatabase> {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open(DB_NAME, DB_VERSION);
    request.onupgradeneeded = () => {
      request.result.createObjectStore(STORE_NAME);
    };
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}

export const idbCache = {
  async get<T>(key: string): Promise<T | undefined> {
    const db = await openDB();
    return new Promise((resolve, reject) => {
      const tx = db.transaction(STORE_NAME, 'readonly');
      const req = tx.objectStore(STORE_NAME).get(key);
      req.onsuccess = () => resolve(req.result as T | undefined);
      req.onerror = () => reject(req.error);
    });
  },

  async set<T>(key: string, value: T): Promise<void> {
    const db = await openDB();
    return new Promise((resolve, reject) => {
      const tx = db.transaction(STORE_NAME, 'readwrite');
      tx.objectStore(STORE_NAME).put(value, key);
      tx.oncomplete = () => resolve();
      tx.onerror = () => reject(tx.error);
    });
  },

  async remove(key: string): Promise<void> {
    const db = await openDB();
    return new Promise((resolve, reject) => {
      const tx = db.transaction(STORE_NAME, 'readwrite');
      tx.objectStore(STORE_NAME).delete(key);
      tx.oncomplete = () => resolve();
      tx.onerror = () => reject(tx.error);
    });
  },
};
```

---

### 6. Cookie Helpers (Consent-Aware)

```typescript
// shared/utils/cookies.ts
interface CookieOptions {
  maxAge?: number;
  path?: string;
  secure?: boolean;
  sameSite?: 'Strict' | 'Lax' | 'None';
}

export function getCookie(name: string): string | null {
  const match = document.cookie.match(new RegExp(`(?:^|; )${name}=([^;]*)`));
  return match ? decodeURIComponent(match[1]) : null;
}

export function setCookie(name: string, value: string, opts: CookieOptions = {}): void {
  const { maxAge = 86400 * 365, path = '/', secure = true, sameSite = 'Lax' } = opts;
  document.cookie = [
    `${name}=${encodeURIComponent(value)}`,
    `max-age=${maxAge}`,
    `path=${path}`,
    secure ? 'secure' : '',
    `samesite=${sameSite}`,
  ].filter(Boolean).join('; ');
}

export function deleteCookie(name: string, path = '/'): void {
  document.cookie = `${name}=; max-age=0; path=${path}`;
}
```

---

### Anti-Patterns

- **Reading storage during render** — causes hydration mismatch; always read in `useEffect` or lazy initializer
- **No namespace isolation** — multi-tenant AEM sites overwrite each other's data
- **Ignoring quota limits** — localStorage is 5-10MB; large data needs IndexedDB
- **Storing sensitive data in localStorage** — tokens belong in httpOnly cookies, not storage
- **No TTL on cached data** — stale content fragments served forever
- **Synchronous IDB calls** — IndexedDB is async-only; wrapping in sync patterns blocks the main thread
- **Cross-tab race conditions** — two tabs writing to the same key without using `StorageEvent` to sync
