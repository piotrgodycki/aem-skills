---
title: AEM Cloud Performance & Caching
impact: CRITICAL
impactDescription: Correct caching configuration can improve page load times by 10x+ and reduce origin load — misconfiguration causes CDN cache misses, stale content, and origin overload
tags: cdn, caching, fastly, cache-control, dispatcher, performance, images, surrogate-control, stale-while-revalidate, auto-scaling
---

## AEM Cloud Performance & Caching

AEM Cloud Service uses Adobe's managed CDN (Fastly). Understanding the multi-tier caching architecture is essential for achieving target TTFB < 200ms and maintaining cache hit ratios above 90%.

---

### 1. CDN Architecture

```
Browser Cache
  ↓ (miss)
Adobe CDN (Fastly) — ~200+ global PoPs
  ↓ (miss)
Origin Shield (Fastly) — single PoP per region
  ↓ (miss)
Dispatcher (Apache httpd) — per-publish pod
  ↓ (miss)
AEM Publish — Java application
```

**Response time targets:**
- CDN hit: 5-50ms
- Origin Shield hit: 50-100ms
- Dispatcher hit: 100-200ms
- AEM Publish render: 200-2000ms (component complexity dependent)

---

### 2. Cache-Control Headers

#### Browser vs CDN Cache

```apache
# Browser cache only (private content)
Cache-Control: max-age=300, private

# Both browser and CDN cache (public content)
Cache-Control: max-age=300, public

# CDN-only cache (Surrogate-Control — stripped before reaching browser)
Surrogate-Control: max-age=3600
Cache-Control: max-age=60
# Browser caches 60s, CDN caches 3600s
```

`Surrogate-Control` applies only to Adobe-managed CDN — stripped before reaching the browser.

#### Content-Type Caching Strategy

| Content Type | Browser TTL | CDN TTL | Strategy |
|-------------|-------------|---------|----------|
| HTML pages | 5 min | 5-60 min | Short TTL + stale-while-revalidate |
| JS/CSS (ClientLibs) | 30 days | 30 days | **Immutable** (hash in URL) |
| DAM images (direct) | 10 min | 12 hours | Moderate TTL |
| Core Image (hashed URL) | 30 days | 30 days | **Immutable** |
| JSON/API responses | 60 sec | 5 min | Short TTL |
| GraphQL persisted queries | 60 sec | 2 hours | CDN-friendly GET |

---

### 3. Caching Configuration (Dispatcher)

#### HTML Pages

```apache
<LocationMatch "^/content/.*\.(html)$">
    Header set Cache-Control "max-age=300,stale-while-revalidate=3600,stale-if-error=86400,public"
    Header set Surrogate-Control "max-age=3600"
    Header set Age 0
</LocationMatch>
```

#### Immutable ClientLib Resources (JS/CSS)

```apache
# Hashed URLs: /etc.clientlibs/mysite/clientlibs/clientlib-site.lc-abc123.min.css
<LocationMatch "^/etc\.clientlibs/.*\.(?i:js|css)$">
    Header set Cache-Control "max-age=2592000,stale-while-revalidate=43200,stale-if-error=43200,public,immutable"
    Header set Age 0
</LocationMatch>
```

#### Mutable ClientLib Resources (Images, Fonts in ClientLibs)

```apache
<LocationMatch "^/etc\.clientlibs/.*\.(?i:json|png|gif|webp|jpe?g|svg|woff2?)$">
    Header set Cache-Control "max-age=43200,stale-while-revalidate=43200,stale-if-error=43200,public"
    Header set Age 0
</LocationMatch>
```

#### Core Image Component (Hashed URLs)

```apache
<LocationMatch "^/content/.*\.coreimg.*\.(?i:jpe?g|png|gif|svg|webp)$">
    Header set Cache-Control "max-age=2592000,stale-while-revalidate=43200,stale-if-error=43200,public,immutable"
    Header set Age 0
</LocationMatch>
```

#### DAM Assets (Direct Access)

```apache
<LocationMatch "^/content/dam/.*\.(?i:jpe?g|gif|js|mov|mp4|png|svg|txt|zip|ico|webp|pdf)$">
    Header set Cache-Control "max-age=43200,stale-while-revalidate=43200,stale-if-error=43200,public"
    Header set Age 0
</LocationMatch>
```

---

### 4. Stale-While-Revalidate & Stale-If-Error

```apache
Cache-Control: max-age=300, stale-while-revalidate=3600, stale-if-error=86400, public
```

| Directive | Behavior | Window |
|-----------|----------|--------|
| `max-age=300` | Content is fresh for 5 min | 0-5 min |
| `stale-while-revalidate=3600` | Serve stale content while fetching fresh in background | 5 min - 65 min |
| `stale-if-error=86400` | Serve stale if origin is down | Up to 24 hours |

**This is the most important caching pattern for AEM.** It provides:
- Fast responses (always serve from cache)
- Fresh content (background revalidation)
- Resilience (origin failures don't affect users)

---

### 5. ClientLib Versioned URLs (Immutable Caching)

```
/etc.clientlibs/mysite/clientlibs/clientlib-site.lc-7c8c5d228445ff48ab49a8e3c865c562-lc.min.css
                                                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                                    Content hash — changes when file changes
```

The hash ensures:
- Browsers always load the latest version after deployment
- Content can be cached for 30+ days (immutable)
- No cache busting query strings needed

**Enable in HTL:**

```html
<sly data-sly-use.clientlib="core/wcm/components/commons/v1/templates/clientlib.html">
    <sly data-sly-call="${clientlib.css @ categories='myproject.site'}"/>
    <sly data-sly-call="${clientlib.js @ categories='myproject.site' loading='async'}"/>
</sly>
```

---

### 6. Cache Invalidation

#### Automatic (Publish/Unpublish)

```
Author publishes page → Sling Content Distribution → Publish receives content
  → Dispatcher flush agent invalidates cached files
  → CDN continues serving until TTL expires (Surrogate-Control)
```

**Important:** CDN is NOT flushed when Dispatcher is invalidated. CDN respects TTLs. This is why short `max-age` + `stale-while-revalidate` is essential for HTML.

#### Programmatic Invalidation

```java
@Reference
private Distributor distributor;

public void invalidateContent(ResourceResolver resolver, String path) {
    DistributionRequest request = new SimpleDistributionRequest(
        DistributionRequestType.INVALIDATE, false, path
    );
    distributor.distribute("publish", resolver, request);
}
```

#### CDN Purge via API (Emergency Only)

```bash
# Available via Cloud Manager API — use sparingly
# Purges should be targeted, never wildcard
aio cloudmanager:purge-cdn --programId=12345 --environmentId=111 \
  --paths="/content/mysite/en.html"
```

---

### 7. Marketing Parameter Stripping

Auto-stripped at CDN layer (since Oct 2023): `utm_*`, `gclid`, `fbclid`, `msclkid`, `_ga`, `mc_*`, `trk_*`, etc.

This prevents cache fragmentation from tracking parameters — the same page with `?utm_source=google` and `?utm_source=facebook` both resolve to the same cached object.

---

### 8. Auto-Scaling and Publish Tier Performance

AEM Cloud Service auto-scales publish pods based on traffic:

```
Low traffic:   2 publish pods (minimum)
Normal:        3-5 pods
High traffic:  10+ pods (auto-scaled)
Peak event:    Pre-scale via Adobe Support request
```

**Implications for frontend:**
- Sticky sessions are NOT used — any pod may serve any request
- Client-side state (sessionStorage) is preferred over server-side state
- Sling jobs may execute on any pod — design for distributed execution

---

### 9. Author Tier Performance

Author performance affects editor experience:

```
Author optimization:
├── Limit components per page (< 100 components)
├── Minimize dialog complexity (fewer XHR calls)
├── Use lazy-loading in dialogs (data-sly-test for heavy panels)
├── Avoid auto-complete fields with large datasets
└── Configure ClientLib minification for author (HTML Library Manager)
```

---

### 10. CDN Hit Ratio Monitoring

Target: **>90% CDN hit ratio** for cacheable content.

```
Monitor via:
├── Cloud Manager → CDN analytics dashboard
├── Fastly real-time stats (Adobe-provided)
├── RUM data (sampleRUM in frontend)
└── Response headers: X-Cache: HIT/MISS, Age: seconds
```

**Debugging cache misses:**

```bash
# Check response headers
curl -I "https://www.mysite.com/en.html"
# Look for:
#   X-Cache: HIT (CDN cache hit)
#   Age: 120 (seconds since cached)
#   Cache-Control: max-age=300, public
#   Vary: Accept-Encoding (only this — nothing else!)
```

---

### 11. Vary Header Configuration

```apache
# CORRECT — only vary on encoding
Vary: Accept-Encoding

# WRONG — these headers fragment the CDN cache
Vary: Accept, Accept-Language, User-Agent, Cookie
# Each unique header combination creates a separate cache entry
```

---

### 12. Asset Optimization

1. **Web-Optimized Image Delivery** — Enable WebP (~25% smaller)
2. **Dynamic Media** — Smart crops, responsive renditions
3. **`_dynamicUrl` in GraphQL** — Server-side image transformation
4. **Lazy loading** — `loading="lazy"` on images below the fold
5. **Preconnect hints** — `<link rel="preconnect" href="https://cdn-host">`
6. **Critical CSS** — Inline above-fold CSS, defer the rest
7. **Avoid large unoptimized images** — never serve original 5000px DAM assets

---

### 13. Anti-Patterns

#### No-Cache on Cacheable Content

```apache
# WRONG — every request hits origin
Cache-Control: no-cache, no-store

# CORRECT — short TTL with background refresh
Cache-Control: max-age=300, stale-while-revalidate=3600, public
```

#### Vary Header Fragmentation

```apache
# WRONG — fragments cache, each combination = separate entry
Vary: Accept, Accept-Language, User-Agent

# CORRECT — only vary on encoding
Vary: Accept-Encoding
```

#### Caching Private Content at CDN

```apache
# WRONG — authenticated content cached at CDN (data leak!)
Cache-Control: max-age=300, public
# This caches user-specific content for ALL users

# CORRECT — private content bypasses CDN
Cache-Control: max-age=300, private
# Or use no-store for sensitive data
```

#### Skipping Hash-Based ClientLib Versioning

```html
<!-- WRONG — no hash, stale CSS after deployment -->
<link rel="stylesheet" href="/etc.clientlibs/mysite/clientlibs/clientlib-site.css">

<!-- CORRECT — hash changes on deploy, browser fetches new version -->
<link rel="stylesheet" href="/etc.clientlibs/mysite/clientlibs/clientlib-site.lc-abc123.min.css">
```

#### Over-Invalidating Cache

```java
// WRONG — flush entire cache on every publish
distributor.distribute("publish", resolver,
    new SimpleDistributionRequest(INVALIDATE, true, "/content")); // ALL content!

// CORRECT — targeted invalidation
distributor.distribute("publish", resolver,
    new SimpleDistributionRequest(INVALIDATE, false, "/content/mysite/en/updated-page"));
```

#### POST GraphQL Queries

```javascript
// WRONG — POST bypasses CDN caching entirely
fetch('/graphql/global/endpoint.json', { method: 'POST', body: query });

// CORRECT — persisted queries use GET → CDN-cacheable
fetch('/graphql/execute.json/mysite/my-query;param=value');
```
