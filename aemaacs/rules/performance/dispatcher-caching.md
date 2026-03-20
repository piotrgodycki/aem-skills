---
title: Dispatcher Caching for AEM Cloud Service
impact: CRITICAL
impactDescription: Proper dispatcher configuration can reduce origin load by 90%+ and improve TTFB by 5-10x for cached content
tags: dispatcher, caching, cache-invalidation, statfileslevel, filter, ttl, sdi, sling-dynamic-include, esi, vanity-urls, cloud-service, apache, cdn
---

## Dispatcher Caching for AEM Cloud Service

The Dispatcher is the Apache HTTP Server-based caching and load-balancing layer between the CDN and AEM Publish. In Cloud Service, the Dispatcher configuration is immutable and deployed via Cloud Manager pipelines.

---

### 1. Dispatcher Architecture in Cloud Service

```
Browser --> Adobe CDN (Fastly) --> Dispatcher (Apache httpd + mod_dispatcher) --> AEM Publish
```

**Cloud Service differences from AEM 6.x:**

| Aspect | AEM 6.x (AMS/On-Prem) | AEM Cloud Service |
|--------|----------------------|-------------------|
| Configuration | Mutable on server | Immutable, deployed via pipeline |
| Apache modules | Many available | Only supported modules |
| Custom scripts | Allowed | Not allowed |
| Dispatcher version | Various | Managed by Adobe |
| Config location | `/etc/httpd/` on server | `dispatcher/src/` in project repo |
| Validation | Manual | Pipeline validation (strict) |

**Project structure:**

```
dispatcher/
  src/
    conf.d/
      available_vhosts/       # Virtual host files
      dispatcher_vhost.conf   # Main vhost config
      rewrites/               # Rewrite rules
    conf.dispatcher.d/
      available_farms/        # Farm configurations
      cache/                  # Cache rules
      clientheaders/          # Allowed client headers
      filters/                # Request filters
      renders/                # Backend AEM instances
      virtualhosts/           # Server name definitions
    opt-in/
      USE_SOURCES_DIRECTLY    # Enables direct source config
```

---

### 2. Cache Rules Configuration

Cache rules define which responses the Dispatcher stores on disk. Rules use glob patterns with `allow` or `deny` types, evaluated in order (last match wins).

**Correct -- selective caching:**

```
# dispatcher/src/conf.dispatcher.d/cache/ams_publish_cache.any
/cache {
    /docroot "/mnt/var/www/html"

    /rules {
        # Deny everything by default
        /0000 { /glob "*" /type "deny" }
        # Allow HTML pages
        /0001 { /glob "*.html" /type "allow" }
        # Allow JSON model exports
        /0002 { /glob "*.json" /type "allow" }
        # Allow client libraries
        /0003 { /glob "/etc.clientlibs/*" /type "allow" }
        # Deny dynamic SDI components
        /0004 { /glob "*.nocache.html*" /type "deny" }
    }
}
```

**What is never cached (regardless of rules):**
- Requests with query strings
- Requests without file extensions
- Responses with `no-cache`, `no-store`, or `must-revalidate` headers
- Responses with `Set-Cookie` headers
- Responses with authentication headers (unless `allowAuthorized` is enabled)

---

### 3. statfileslevel and Cache Invalidation Granularity

`statfileslevel` controls how granularly cache invalidation works by creating `.stat` files at directory levels from the docroot.

**How it works:**

```
/statfileslevel "2"

# Creates .stat files at:
# /mnt/var/www/html/.stat                    (level 0 - docroot)
# /mnt/var/www/html/content/.stat            (level 1)
# /mnt/var/www/html/content/mysite/.stat     (level 2)
```

When content at `/content/mysite/en/page.html` is invalidated, the `.stat` file at level 2 (`/content/mysite/.stat`) is touched. Only cached files under `/content/mysite/` with a modification time older than the `.stat` file are refetched.

**Incorrect -- statfileslevel too low:**

```
/statfileslevel "0"
# ANY content change touches the root .stat file
# ENTIRE cache is invalidated on every publish action
# Result: cache miss storm after every activation
```

**Correct -- statfileslevel matching site structure:**

```
# For multi-site: /content/site-a/en, /content/site-b/fr
/statfileslevel "3"
# Publishing to site-a only invalidates site-a's language subtree
# site-b remains fully cached
```

**Rule of thumb**: Set `statfileslevel` to match the depth where independent site content trees diverge. For `/content/{site}/{language}/...` structures, use `3`.

---

### 4. Filter Configuration

Filters control which HTTP requests reach AEM Publish. Use an allowlist approach: deny everything, then selectively allow.

**Correct -- secure filter configuration:**

```
# dispatcher/src/conf.dispatcher.d/filters/filters.any
/filter {
    # Deny all requests by default
    /0001 { /type "deny" /url "*" }

    # Allow page content
    /0010 { /type "allow" /url "/content/*" /extension '(html|json)' }

    # Allow client libraries
    /0020 { /type "allow" /url "/etc.clientlibs/*" }

    # Allow DAM assets
    /0030 { /type "allow" /url "/content/dam/*"
            /extension '(jpg|jpeg|png|gif|svg|webp|pdf|zip)' }

    # Allow GraphQL persisted queries
    /0040 { /type "allow" /url "/graphql/execute.json/*" /method "GET" }

    # Allow content model exports
    /0050 { /type "allow" /url "/content/*" /extension "model.json" }

    # Block sensitive paths
    /0100 { /type "deny" /url "/bin/*" }
    /0101 { /type "deny" /url "/system/*" }
    /0102 { /type "deny" /url "/apps/*" }
    /0103 { /type "deny" /url "/libs/*" }
    /0104 { /type "deny" /url "/crx/*" }

    # Allow specific servlets if needed
    /0200 { /type "allow" /url "/bin/mysite/articles" /method "GET" }
}
```

**Important**: Glob-based filtering is deprecated. Use the decomposed URL elements (`/url`, `/method`, `/extension`, `/selector`, `/suffix`) for security.

**Incorrect -- overly permissive filter:**

```
# NEVER do this -- exposes internal AEM paths
/0001 { /type "allow" /url "*" }
/0002 { /type "deny" /url "/crx/*" }
# Attacker can access /system/console, /bin/querybuilder.json, etc.
```

---

### 5. Header Caching and TTL Configuration

#### Cached Response Headers

```
/cache {
    /headers {
        "Cache-Control"
        "Content-Disposition"
        "Content-Type"
        "Expires"
        "Last-Modified"
        "X-Content-Type-Options"
    }
}
```

#### TTL-Based Caching (enableTTL)

When enabled, the Dispatcher respects `Cache-Control` max-age and `Expires` headers for cache lifetime instead of relying solely on `.stat` file invalidation.

```
/cache {
    /enableTTL "1"
}
```

**Behavior with enableTTL (Dispatcher 4.3.5+):**
1. TTL expiration checked first
2. If not expired, standard `.stat` invalidation rules still apply
3. Balances content freshness over cache-hit ratio

#### Setting TTLs per Content Type (vhost configuration)

```apache
# dispatcher/src/conf.d/available_vhosts/default.vhost

# HTML pages: short TTL, background refresh
<LocationMatch "^/content/.*\.html$">
    Header set Cache-Control "max-age=300,stale-while-revalidate=3600,stale-if-error=86400,public"
    Header set Surrogate-Control "max-age=3600"
    Header set Age 0
</LocationMatch>

# JSON model exports: medium TTL
<LocationMatch "^/content/.*\.model\.json$">
    Header set Cache-Control "max-age=600,stale-while-revalidate=1800,public"
    Header set Age 0
</LocationMatch>

# Client libraries (versioned): long TTL, immutable
<LocationMatch "^/etc\.clientlibs/.*\.lc-.*\.(css|js)$">
    Header set Cache-Control "max-age=2592000,public,immutable"
    Header set Age 0
</LocationMatch>

# DAM assets: medium TTL
<LocationMatch "^/content/dam/.*\.(?i:jpe?g|png|gif|svg|webp|pdf)$">
    Header set Cache-Control "max-age=43200,stale-while-revalidate=43200,stale-if-error=43200,public"
    Header set Age 0
</LocationMatch>

# Web fonts: long TTL
<LocationMatch "^/etc\.clientlibs/.*\.(woff2?|ttf|eot)$">
    Header set Cache-Control "max-age=2592000,public,immutable"
    Header set Age 0
</LocationMatch>
```

**Key header distinction:**
- `Cache-Control` -- controls both browser and CDN cache duration
- `Surrogate-Control` -- controls CDN only (stripped before reaching browser)
- Use `Surrogate-Control` when CDN should cache longer than the browser

---

### 6. Dispatcher Flush and Automatic Invalidation

#### Automatic Invalidation

When content is published, AEM sends invalidation requests to the Dispatcher. Configure which file types are invalidated:

```
/cache {
    /invalidate {
        /0000 { /glob "*" /type "deny" }
        /0001 { /glob "*.html" /type "allow" }
        /0002 { /glob "*.json" /type "allow" }
    }
}
```

When a page at `/content/mysite/en/page.html` is published:
1. The cached `.html` file is deleted
2. The `.stat` file at the appropriate level is touched
3. Other cached files with older timestamps than `.stat` are refetched on next request
4. The CDN is **not** flushed -- it respects its own TTLs

#### Grace Period

Prevents cache-miss storms during batch activations:

```
/cache {
    /gracePeriod "2"
}
```

During the grace period (in seconds), invalidated content may still be served from cache. This prevents repeated origin hits when multiple pages are published simultaneously.

---

### 7. Caching Personalized and Authenticated Content

#### Permission-Sensitive Caching

Dispatchers can cache content for authenticated users when configured with Permission Sensitive Caching (PSC):

```
/cache {
    /allowAuthorized "1"
}
```

**Warning**: Only use this when combined with AEM's permission check mechanism. The Dispatcher sends a HEAD request to AEM to verify access before serving cached content.

#### TTL-Based Approach for Semi-Dynamic Content

For content that is mostly static but has a personalized header/greeting:

```apache
# Cache the main page structure with short TTL
<LocationMatch "^/content/mysite/.*\.html$">
    Header set Cache-Control "max-age=60,public"
    Header set Age 0
</LocationMatch>
```

Combine with client-side personalization (fetch user-specific data via AJAX after page load).

#### Sling Dynamic Include for Partially Cacheable Pages

See section 13 below for SDI patterns that allow caching the page while excluding dynamic components.

---

### 8. Cache-Busting Patterns

#### Versioned ClientLib URLs (Built-in)

AEM automatically generates content-hash URLs for client libraries:

```
/etc.clientlibs/mysite/clientlibs/clientlib-site.lc-7c8c5d228445ff48ab49a8e3c865c562-lc.css
```

The `.lc-{hash}-lc.` segment changes when file content changes, enabling immutable caching.

#### Versioned Image URLs (Core Image Component)

```
/content/mysite/en/page/_jcr_content/root/image.coreimg.85.1200.jpeg/1709123456789/hero.jpeg
```

The timestamp segment (`1709123456789`) changes on re-publish, busting CDN and browser caches.

#### Custom Cache-Busting for DAM Assets

**Incorrect -- query parameter cache busting:**

```html
<!-- Query parameters are NOT cached by Dispatcher -->
<img src="/content/dam/mysite/hero.jpg?v=12345" />
<!-- This ALWAYS misses the Dispatcher cache -->
```

**Correct -- selector-based or path-based versioning:**

```html
<!-- Use the Core Image Component which handles versioning automatically -->
<div data-sly-use.image="com.adobe.cq.wcm.core.components.models.Image">
    <img src="${image.src}" />
    <!-- Outputs: /content/.../image.coreimg.85.1200.jpeg/1709123456789/hero.jpeg -->
</div>
```

---

### 9. Dispatcher Rewrites for Vanity URLs

```apache
# dispatcher/src/conf.d/rewrites/rewrite.rules

RewriteEngine On

# Vanity URL: /about -> /content/mysite/en/about-us.html
RewriteRule ^/about$ /content/mysite/en/about-us.html [PT,L]

# Short URL pattern: /blog/my-article -> /content/mysite/en/blog/my-article.html
RewriteCond %{REQUEST_URI} !^/content/
RewriteCond %{REQUEST_URI} !^/etc/
RewriteCond %{REQUEST_URI} !^/libs/
RewriteRule ^/(.*)$ /content/mysite/en/$1.html [PT,L]

# Language-based redirect
RewriteCond %{HTTP:Accept-Language} ^de [NC]
RewriteRule ^/$ /content/mysite/de.html [R=302,L]
```

**Caching consideration**: Vanity URL rewrites happen at the Apache level, so the Dispatcher caches the resolved (internal) path. Both `/about` and `/content/mysite/en/about-us.html` serve the same cached file.

---

### 10. .stat File Mechanism

The `.stat` file is a zero-byte marker file whose modification timestamp determines cache validity.

**Invalidation flow:**

```
1. Author publishes /content/mysite/en/news/article.html
2. Replication agent sends invalidation to Dispatcher
3. Dispatcher touches .stat at configured level:
   /mnt/var/www/html/content/mysite/en/.stat (if statfileslevel=3)
4. Next request for any file under /content/mysite/en/:
   - Dispatcher compares file mtime vs .stat mtime
   - If file is older than .stat -> re-fetch from AEM
   - If file is newer than .stat -> serve from cache
```

**serveStaleOnError**: Serves stale cached content when AEM Publish is unavailable:

```
/cache {
    /serveStaleOnError "1"
}
```

---

### 11. Cache Headers for Different Content Types

| Content Type | Cache-Control | Surrogate-Control | Typical TTL |
|-------------|---------------|-------------------|-------------|
| HTML pages | `max-age=300,stale-while-revalidate=3600,public` | `max-age=3600` | 5 min browser, 1 hr CDN |
| JSON exports | `max-age=600,stale-while-revalidate=1800,public` | -- | 10 min |
| JS/CSS (versioned) | `max-age=2592000,public,immutable` | -- | 30 days |
| JS/CSS (unversioned) | `max-age=43200,public` | -- | 12 hours |
| DAM images | `max-age=43200,stale-while-revalidate=43200,public` | -- | 12 hours |
| Core Component images | `max-age=2592000,public,immutable` | -- | 30 days |
| Web fonts | `max-age=2592000,public,immutable` | -- | 30 days |
| GraphQL persisted | `max-age=7200,stale-while-revalidate=3600,public` | -- | 2 hours |
| Personalized content | `private,no-store` | -- | No cache |

---

### 12. Debugging Dispatcher Cache

#### X-Dispatcher Headers

Enable diagnostic headers in the farm configuration:

```
/farm {
    /info {
        /header "X-Dispatcher-Info"
    }
}
```

**Response headers to check:**

| Header | Meaning |
|--------|---------|
| `X-Dispatcher: <version>` | Confirms Dispatcher handled request |
| `X-Cache: HIT` | Served from Dispatcher cache |
| `X-Cache: MISS` | Fetched from AEM Publish |
| `X-Cache-Info: caching` | Response was cached |
| `X-Cache-Info: not cacheable: ...` | Reason for not caching |

#### Common Cache Miss Causes

| Cause | Diagnosis | Fix |
|-------|-----------|-----|
| Query string in URL | Check `X-Cache-Info` | Configure `ignoreUrlParams` |
| Missing extension | URL has no `.html`/`.json` | Add extension or configure rule |
| Authentication header | `Authorization` header present | Remove or use `allowAuthorized` |
| Set-Cookie in response | Response sets cookies | Remove cookies from cacheable responses |
| `no-cache` header | Response has `Cache-Control: no-cache` | Fix header in vhost or AEM code |
| Not in cache rules | Path not matched by `/rules` | Add allow rule |

#### Ignore URL Parameters

Prevent marketing/tracking parameters from fragmenting the cache:

```
/cache {
    /ignoreUrlParams {
        /0001 { /glob "*" /type "deny" }
        /0002 { /glob "utm_*" /type "allow" }
        /0003 { /glob "gclid" /type "allow" }
        /0004 { /glob "fbclid" /type "allow" }
        /0005 { /glob "msclkid" /type "allow" }
    }
}
```

Note: In Cloud Service, the CDN (Fastly) automatically strips common marketing parameters (`utm_*`, `gclid`, `fbclid`, `msclkid`, `_ga`, etc.) before they reach the Dispatcher.

---

### 13. Sling Dynamic Include (SDI) for Partially Cacheable Pages

SDI replaces component includes with server-side includes (SSI), allowing the page to be cached while dynamic components are fetched separately.

#### OSGi Configuration

```xml
<!-- ui.apps/src/main/content/jcr_root/apps/mysite/config.publish/
     org.apache.sling.dynamicinclude.Configuration-mysite.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
    xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    jcr:primaryType="sling:OsgiConfig"
    include-filter.config.enabled="{Boolean}true"
    include-filter.config.path="/content/mysite"
    include-filter.config.resource-types="[mysite/components/header/user-greeting,mysite/components/cart-count]"
    include-filter.config.include-type="SSI"
    include-filter.config.selector="nocache"
    include-filter.config.ttl=""
    include-filter.config.required_header="Server-Agent=Communique-Dispatcher"
    include-filter.config.rewrite="{Boolean}true"/>
```

#### Apache Configuration (vhost)

```apache
# Enable SSI processing on HTML files
<Directory /mnt/var/www/html>
    Options FollowSymLinks Includes
    AddOutputFilter INCLUDES .html
</Directory>
```

Note: `mod_include` is loaded by default in AEM Cloud Service.

#### Dispatcher Cache Rules for SDI

```
/cache {
    /rules {
        # Cache HTML pages
        /0001 { /glob "*.html" /type "allow" }
        # DO NOT cache nocache selector (dynamic components)
        /0002 { /glob "*.nocache.html*" /type "deny" }
    }
}
```

#### How It Works

1. Page renders normally in AEM
2. SDI filter replaces targeted component output with SSI directives:
   ```html
   <!--#include virtual="/content/mysite/en/page/jcr:content/header/user-greeting.nocache.html" -->
   ```
3. Dispatcher caches the page with SSI directives (not the component output)
4. Apache processes SSI directives on each request, fetching dynamic components from AEM
5. Dynamic component responses are NOT cached (due to `nocache` deny rule)

#### Include Type Options

| Type | Where Processed | Use Case |
|------|----------------|----------|
| SSI | Apache (Dispatcher) | Default choice, works in Cloud Service |
| ESI | CDN edge | Better for high-traffic sites (requires CDN support) |
| JSI | Client browser (AJAX) | Fallback when SSI/ESI not available |

**ESI in Cloud Service**: Edge Side Includes are supported via the CDN. Configure with `include-filter.config.include-type="ESI"` and ensure the CDN is configured to process ESI tags.

---

### 14. Common Mistakes

#### Caching Query Parameters

**Incorrect:**

```html
<!-- Links with tracking parameters fragment the cache -->
<a href="/content/mysite/en/page.html?source=email&campaign=spring">Link</a>
<!-- Each unique query string creates a separate cache entry (or misses entirely) -->
```

**Correct:**

Configure `ignoreUrlParams` in the Dispatcher and handle tracking client-side:

```javascript
// Move tracking to client-side JS instead of URL parameters
document.addEventListener('DOMContentLoaded', () => {
    if (window.location.search.includes('source=email')) {
        analytics.track('email-referral', { campaign: 'spring' });
    }
});
```

#### Missing statfileslevel

**Incorrect:**

```
# No statfileslevel configured (defaults to 0)
# Publishing ANY page invalidates the ENTIRE cache
```

**Correct:**

```
/statfileslevel "3"
# Invalidation scoped to the site/language subtree
```

#### Overly Broad Invalidation

**Incorrect:**

```
/invalidate {
    /0000 { /glob "*" /type "allow" }
    # Every activation invalidates ALL cached file types
    # JS, CSS, images all re-fetched unnecessarily
}
```

**Correct:**

```
/invalidate {
    /0000 { /glob "*" /type "deny" }
    /0001 { /glob "*.html" /type "allow" }
    /0002 { /glob "*.json" /type "allow" }
    # Only HTML and JSON invalidated -- static assets remain cached
}
```

#### Not Using stale-while-revalidate

**Incorrect:**

```apache
# Hard TTL with no grace -- users see slow responses when cache expires
Header set Cache-Control "max-age=300"
```

**Correct:**

```apache
# Background refresh -- users always get fast response
Header set Cache-Control "max-age=300,stale-while-revalidate=3600,stale-if-error=86400,public"
```

---

### 15. Dispatcher Converter Tool for Cloud Migration

When migrating from AEM 6.x (AMS or on-premise) to Cloud Service, use the **Dispatcher Converter** tool to transform existing configurations to the Cloud Service format:

- Converts legacy `dispatcher.any` to the modular Cloud Service structure
- Identifies unsupported modules and configurations
- Validates against Cloud Service constraints
- Part of the AEM Modernization Tools suite

Run as part of Cloud Readiness analysis in Cloud Manager.

---

### 16. Testing Dispatcher Locally with Docker SDK

The AEM Cloud Service SDK includes a Docker-based Dispatcher image for local testing:

```bash
# Download the Dispatcher SDK from Software Distribution portal
# Extract and run the validation script
./bin/validator full dispatcher/src

# Start local Dispatcher with Docker
./bin/docker_run.sh dispatcher/src host.docker.internal:4503 8080

# Test cached responses
curl -I http://localhost:8080/content/mysite/en.html
# Check for X-Dispatcher and X-Cache headers
```

**Validation checks:**
- Configuration syntax errors
- Unsupported modules or directives
- Filter security (overly permissive rules)
- File structure compliance

**Local testing workflow:**
1. Edit config in `dispatcher/src/`
2. Run `validator full` to check syntax
3. Start Docker Dispatcher pointing to local AEM SDK Publish
4. Test cache behavior with `curl -I` to inspect headers
5. Verify invalidation by publishing content and checking cache misses

---

### Pitfalls

- Setting `statfileslevel` to 0 (invalidates entire cache on any publish)
- Using glob-based filters instead of decomposed URL elements (security risk)
- Caching responses that set cookies (prevents all subsequent caching)
- Not configuring `ignoreUrlParams` for marketing/tracking parameters
- Forgetting `enableTTL "1"` when relying on Cache-Control headers for Dispatcher TTL
- Using `allowAuthorized` without Permission Sensitive Caching checks
- Adding `Vary` headers beyond `Accept-Encoding` (fragments CDN cache)
- Not testing Dispatcher config locally before pipeline deployment (pipeline failures)
- Assuming CDN is flushed when Dispatcher cache is invalidated (CDN respects its own TTLs)
- Blocking `/bin/*` without whitelisting needed custom servlets
- Missing `serveStaleOnError` -- causes downtime when AEM Publish is temporarily unavailable
