---
title: Service Workers for EDS
impact: MEDIUM
impactDescription: Service workers enable offline support and advanced caching — but incorrect patterns break EDS push invalidation and degrade rather than improve performance
tags: eds, service-worker, offline, caching, workbox, precache, background-sync, push-notifications
---

## Service Workers for EDS

Service workers intercept network requests, enabling offline support, advanced caching, and background sync. In EDS, the CDN already provides excellent caching with push invalidation — service workers add value for offline-first experiences and advanced use cases, but must not conflict with EDS's caching model.

---

### 1. When to Use Service Workers with EDS

| Use Case | Recommended | Notes |
|----------|------------|-------|
| Offline fallback page | Yes | Show custom offline page instead of browser error |
| Precaching app shell | Cautious | Only for app-like experiences, not content sites |
| Form background sync | Yes | Queue form submissions when offline |
| Push notifications | Yes | Re-engagement (requires user consent) |
| Full offline app | Cautious | Significant complexity; consider if truly needed |
| Caching content pages | **No** | EDS CDN handles this better with push invalidation |

**Critical rule:** Never cache HTML responses from EDS in a service worker — EDS uses push invalidation to purge stale content from CDN. A service worker cache would serve stale content indefinitely.

---

### 2. Registration

Register in `delayed.js` to avoid impacting LCP:

```javascript
// scripts/delayed.js
export default function loadDelayed() {
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js', { scope: '/' })
      .then((registration) => {
        console.log('SW registered:', registration.scope);
      })
      .catch((error) => {
        console.error('SW registration failed:', error);
      });
  }
}
```

---

### 3. Basic Service Worker (Offline Fallback)

```javascript
// /sw.js — placed at site root
const CACHE_NAME = 'eds-offline-v1';
const OFFLINE_PAGE = '/offline.html';

// Precache the offline page on install
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.add(OFFLINE_PAGE))
  );
  self.skipWaiting();
});

// Clean up old caches on activate
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(keys.filter((key) => key !== CACHE_NAME).map((key) => caches.delete(key)))
    )
  );
  self.clients.claim();
});

// Network-first for navigation, fallback to offline page
self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => caches.match(OFFLINE_PAGE))
    );
  }
  // Let all other requests go to network (EDS CDN handles caching)
});
```

---

### 4. Caching Strategies

#### Network-First (Recommended for EDS)

```javascript
// Only for non-HTML assets that benefit from offline access
async function networkFirst(request, cacheName) {
  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open(cacheName);
      cache.put(request, response.clone());
    }
    return response;
  } catch {
    return caches.match(request);
  }
}
```

#### Cache-First (Static Assets Only)

```javascript
// Only for versioned/hashed static assets (fonts, icons)
async function cacheFirst(request, cacheName) {
  const cached = await caches.match(request);
  if (cached) return cached;

  const response = await fetch(request);
  if (response.ok) {
    const cache = await caches.open(cacheName);
    cache.put(request, response.clone());
  }
  return response;
}
```

#### Strategy Router

```javascript
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // Navigation — network only, offline fallback
  if (request.mode === 'navigate') {
    event.respondWith(
      fetch(request).catch(() => caches.match(OFFLINE_PAGE))
    );
    return;
  }

  // Fonts — cache-first (rarely change)
  if (url.pathname.match(/\.(woff2?|ttf|otf)$/)) {
    event.respondWith(cacheFirst(request, 'fonts-v1'));
    return;
  }

  // Images — network-first with offline fallback
  if (url.pathname.match(/\.(png|jpg|jpeg|webp|avif|svg)$/)) {
    event.respondWith(networkFirst(request, 'images-v1'));
    return;
  }

  // Everything else — network only (let EDS CDN handle it)
});
```

---

### 5. Background Sync for Forms

Queue form submissions when offline:

```javascript
// In your form block JS
async function submitForm(formData) {
  try {
    const response = await fetch('/form-endpoint', {
      method: 'POST',
      body: formData,
    });
    return response.json();
  } catch {
    // Queue for background sync
    if ('serviceWorker' in navigator && 'SyncManager' in window) {
      const registration = await navigator.serviceWorker.ready;
      await saveFormData(formData); // Save to IndexedDB
      await registration.sync.register('form-submit');
      return { queued: true, message: 'Saved — will submit when online' };
    }
    throw new Error('Offline and no sync support');
  }
}
```

```javascript
// In sw.js
self.addEventListener('sync', (event) => {
  if (event.tag === 'form-submit') {
    event.waitUntil(submitQueuedForms());
  }
});

async function submitQueuedForms() {
  const db = await openDB();
  const forms = await db.getAll('pending-forms');

  for (const form of forms) {
    try {
      await fetch('/form-endpoint', {
        method: 'POST',
        body: JSON.stringify(form.data),
        headers: { 'Content-Type': 'application/json' },
      });
      await db.delete('pending-forms', form.id);
    } catch {
      // Will retry on next sync event
      break;
    }
  }
}
```

---

### 6. Cache Versioning and Cleanup

```javascript
const CACHE_VERSION = 'v2';
const CACHE_NAMES = {
  offline: `eds-offline-${CACHE_VERSION}`,
  fonts: `eds-fonts-${CACHE_VERSION}`,
  images: `eds-images-${CACHE_VERSION}`,
};

self.addEventListener('activate', (event) => {
  const validCaches = new Set(Object.values(CACHE_NAMES));
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys.filter((key) => !validCaches.has(key)).map((key) => caches.delete(key))
      )
    )
  );
  self.clients.claim();
});
```

---

### 7. Anti-Patterns

#### Caching HTML Pages

```javascript
// WRONG — caching EDS HTML breaks push invalidation
self.addEventListener('fetch', (event) => {
  if (request.mode === 'navigate') {
    event.respondWith(cacheFirst(request, 'pages')); // Stale content forever!
  }
});

// CORRECT — never cache HTML, only provide offline fallback
self.addEventListener('fetch', (event) => {
  if (request.mode === 'navigate') {
    event.respondWith(
      fetch(request).catch(() => caches.match(OFFLINE_PAGE))
    );
  }
});
```

#### Caching Everything

```javascript
// WRONG — blanket caching defeats EDS CDN and wastes storage
self.addEventListener('fetch', (event) => {
  event.respondWith(cacheFirst(event.request, 'everything'));
});

// CORRECT — selective caching of specific asset types only
```

#### Registering SW Eagerly

```javascript
// WRONG — SW registration in head or scripts.js blocks main thread
// <script>navigator.serviceWorker.register('/sw.js')</script>

// CORRECT — register in delayed.js (after page is interactive)
// See section 2
```

#### Not Handling SW Updates

```javascript
// WRONG — no update mechanism, users stuck on old SW forever

// CORRECT — skipWaiting + clients.claim for immediate activation
self.addEventListener('install', () => self.skipWaiting());
self.addEventListener('activate', (event) => {
  event.waitUntil(self.clients.claim());
});
```
