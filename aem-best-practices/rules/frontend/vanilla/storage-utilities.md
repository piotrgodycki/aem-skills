---
title: Storage Utilities for Vanilla JS AEM
impact: MEDIUM
impactDescription: Unmanaged storage causes cross-site data leaks in multi-tenant AEM, quota errors, and stale cached content
tags: vanilla-js, storage, localStorage, sessionStorage, indexeddb, cache, ttl, namespace
---

## Storage Utilities for Vanilla JS AEM

A storage abstraction layer for AEM vanilla JS components — namespaced by site, TTL-based expiration, quota management, and a fallback chain from memory to IndexedDB.

---

### 1. Namespaced Storage with TTL

```typescript
// utils/storage.ts
interface StorageEntry<T> {
  v: T;          // value
  e?: number;    // expires timestamp
}

export class SiteStorage {
  private prefix: string;
  private backend: Storage;

  constructor(siteId: string, backend: Storage = localStorage) {
    this.prefix = `${siteId}:`;
    this.backend = backend;
  }

  get<T>(key: string, fallback: T): T {
    try {
      const raw = this.backend.getItem(this.prefix + key);
      if (!raw) return fallback;

      const entry: StorageEntry<T> = JSON.parse(raw);
      if (entry.e && Date.now() > entry.e) {
        this.backend.removeItem(this.prefix + key);
        return fallback;
      }
      return entry.v;
    } catch {
      return fallback;
    }
  }

  set<T>(key: string, value: T, ttlMs?: number): boolean {
    const entry: StorageEntry<T> = { v: value };
    if (ttlMs) entry.e = Date.now() + ttlMs;

    try {
      this.backend.setItem(this.prefix + key, JSON.stringify(entry));
      return true;
    } catch {
      // Quota exceeded — try eviction
      this.evictExpired();
      try {
        this.backend.setItem(this.prefix + key, JSON.stringify(entry));
        return true;
      } catch {
        return false;
      }
    }
  }

  remove(key: string): void {
    this.backend.removeItem(this.prefix + key);
  }

  clear(): void {
    const toRemove: string[] = [];
    for (let i = 0; i < this.backend.length; i++) {
      const k = this.backend.key(i);
      if (k?.startsWith(this.prefix)) toRemove.push(k);
    }
    toRemove.forEach(k => this.backend.removeItem(k));
  }

  private evictExpired(): void {
    const now = Date.now();
    for (let i = this.backend.length - 1; i >= 0; i--) {
      const k = this.backend.key(i);
      if (!k?.startsWith(this.prefix)) continue;
      try {
        const entry = JSON.parse(this.backend.getItem(k)!);
        if (entry.e && now > entry.e) this.backend.removeItem(k);
      } catch {
        // Corrupted entry — remove
        this.backend.removeItem(k!);
      }
    }
  }
}

// Per-site instances
export const store = new SiteStorage('mysite');
export const session = new SiteStorage('mysite', sessionStorage);
```

---

### 2. Fallback Chain

When primary storage fails (private browsing, quota), fall back gracefully:

```typescript
// utils/storage-chain.ts
interface StorageAdapter {
  get<T>(key: string): T | undefined;
  set<T>(key: string, value: T): boolean;
  remove(key: string): void;
}

class MemoryStorage implements StorageAdapter {
  private data = new Map<string, unknown>();

  get<T>(key: string): T | undefined {
    return this.data.get(key) as T | undefined;
  }

  set<T>(key: string, value: T): boolean {
    this.data.set(key, value);
    return true;
  }

  remove(key: string): void {
    this.data.delete(key);
  }
}

class LocalStorageAdapter implements StorageAdapter {
  private prefix: string;
  constructor(prefix: string) { this.prefix = prefix; }

  get<T>(key: string): T | undefined {
    try {
      const raw = localStorage.getItem(this.prefix + key);
      return raw ? JSON.parse(raw) : undefined;
    } catch {
      return undefined;
    }
  }

  set<T>(key: string, value: T): boolean {
    try {
      localStorage.setItem(this.prefix + key, JSON.stringify(value));
      return true;
    } catch {
      return false;
    }
  }

  remove(key: string): void {
    localStorage.removeItem(this.prefix + key);
  }
}

export function createStorageChain(prefix: string): StorageAdapter {
  const local = new LocalStorageAdapter(prefix);
  const memory = new MemoryStorage();

  return {
    get<T>(key: string): T | undefined {
      return local.get<T>(key) ?? memory.get<T>(key);
    },
    set<T>(key: string, value: T): boolean {
      if (local.set(key, value)) return true;
      return memory.set(key, value); // fallback to memory
    },
    remove(key: string): void {
      local.remove(key);
      memory.remove(key);
    },
  };
}

export const storage = createStorageChain('mysite:');
```

---

### 3. Usage in AEM Components

```typescript
// components/search/search.ts
import { session } from '@utils/storage';

const RECENT_SEARCHES_KEY = 'recent-searches';
const MAX_RECENT = 5;

export function getRecentSearches(): string[] {
  return session.get<string[]>(RECENT_SEARCHES_KEY, []);
}

export function addRecentSearch(term: string): void {
  const recent = getRecentSearches().filter(t => t !== term);
  recent.unshift(term);
  session.set(RECENT_SEARCHES_KEY, recent.slice(0, MAX_RECENT));
}

// components/theme/theme.ts
import { store } from '@utils/storage';

export function getThemePreference(): 'light' | 'dark' {
  return store.get('theme', 'light');
}

export function setThemePreference(theme: 'light' | 'dark'): void {
  store.set('theme', theme);
  document.documentElement.dataset.theme = theme;
}
```

---

### 4. Quota Management

```typescript
// utils/storage-quota.ts
export function getStorageUsage(prefix: string): { used: number; items: number } {
  let used = 0;
  let items = 0;

  for (let i = 0; i < localStorage.length; i++) {
    const key = localStorage.key(i);
    if (!key?.startsWith(prefix)) continue;
    const value = localStorage.getItem(key) || '';
    used += key.length + value.length;
    items++;
  }

  return { used: used * 2, items }; // UTF-16 = 2 bytes per char
}

export function estimateQuota(): number {
  // Most browsers: 5MB for localStorage
  return 5 * 1024 * 1024;
}

// Check before writing large data
const { used } = getStorageUsage('mysite:');
const available = estimateQuota() - used;
if (available < dataSize) {
  // Use IndexedDB instead
}
```

---

### Anti-Patterns

- **No namespace prefix** — multi-tenant AEM sites overwrite each other's data
- **Storing without TTL** — cached API responses served forever even when content is republished
- **Not handling quota errors** — `setItem` throws `QuotaExceededError` in private browsing and when full
- **Storing PII in localStorage** — use httpOnly cookies for auth tokens; localStorage is XSS-accessible
- **Parsing without try/catch** — corrupted JSON crashes the entire component
- **Using localStorage in SSR** — always check `typeof window !== 'undefined'` first
