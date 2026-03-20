---
title: CDN Configuration for AEM Edge Delivery Services
impact: HIGH
impactDescription: EDS CDN configuration directly controls caching, invalidation, redirects, and security headers — mistakes cause stale content, broken domains, or security vulnerabilities
tags: eds, cdn, caching, custom-domain, redirects, push-invalidation, headers, cors, csp, byo-cdn, aem-live, performance
---

## CDN Configuration for AEM Edge Delivery Services

Edge Delivery Services uses its own CDN infrastructure (the aem.live / hlx.live delivery network) that is distinct from the Fastly CDN used by AEMaaCS Publish. Content flows through a push-based invalidation model where publishing automatically purges the CDN.

---

### 1. EDS CDN Architecture

#### 1.1 Four-Layer Stack

```
1. Customer infrastructure    (DNS, TLS, optional BYO CDN)
2. Adobe edge compute layer   (aem.live CDN, edge functions)
3. Adobe storage layer        (Content Bus, Media Bus, Code Bus)
4. Customer content sources   (SharePoint, Google Drive, AEM Author)
```

#### 1.2 Three-Tier URL Structure

| Tier | Domain Pattern | Purpose |
|------|---------------|---------|
| Preview | `https://main--<site>--<org>.aem.page` | Author preview (low cache TTL) |
| Live | `https://main--<site>--<org>.aem.live` | Production CDN (higher cache TTL) |
| Custom | `https://www.example.com` | CNAME to `.aem.live` |

Legacy domains (`.hlx.page` / `.hlx.live`) continue to work but `.aem.page` / `.aem.live` are the current standard.

Branch-based environments: every Git branch gets its own preview and live URLs (e.g., `feature-123--mysite--myorg.aem.page`).

#### 1.3 Content Flow

```
Author edits in SharePoint/Google Drive/AEM Author
  --> Sidekick "Preview" --> Content pulled into Adobe storage (Content Bus)
  --> Sidekick "Publish" --> Content pushed to live CDN + cache purged across all layers
```

Publishing triggers **automatic, surgical cache purging** at every caching layer, including the CDN. There is no TTL-based stale window for published content -- purging is push-based.

#### 1.4 Media Handling

Media assets use content-addressable storage with hash-based URLs:

```
/media_1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1.png
```

The hash is derived from the binary content of the asset, making these URLs **permanently cacheable** (immutable). If the asset content changes, a new hash (new URL) is generated.

---

### 2. Custom Domain Setup

#### 2.1 Adobe-Managed CDN (Recommended)

The simplest path uses Adobe's managed CDN with DNS pointing directly to Adobe infrastructure.

**DNS configuration:**

| Domain Type | Record Type | Value |
|-------------|-------------|-------|
| Subdomain (www) | CNAME | `cdn.adobeaemcloud.com` |
| Apex domain | A records | `151.101.3.10`, `151.101.67.10`, `151.101.131.10`, `151.101.195.10` |

Apex domain A records automatically redirect non-www to www.

**SSL/TLS:** Certificates are provisioned automatically (Let's Encrypt) or you can bring your own certificate. HTTPS is mandatory. Configure both `www` and apex variants for proper certificate generation.

**Recommended DNS TTL:** 3600 seconds (1 hour). Lower temporarily before go-live to speed up propagation.

**Push invalidation configuration:**

Add these properties to your site configuration spreadsheet (`.helix/config.xlsx` in SharePoint or `.helix/config` in Google Drive):

| Key | Value |
|-----|-------|
| `cdn.prod.host` | `www.example.com` |
| `cdn.prod.type` | `managed` |

Preview and activate the configuration via Sidekick to apply.

#### 2.2 BYO CDN (Bring Your Own CDN)

EDS supports placing a customer-managed CDN (Cloudflare, Akamai, Fastly, Amazon CloudFront) in front of the EDS origin.

**Origin URL format:**

```
https://main--<yoursite>--<yourorg>.aem.live
```

**Required headers from your CDN to EDS origin:**

| Header | Value | Purpose |
|--------|-------|---------|
| `X-Forwarded-Host` | Production domain (e.g., `www.example.com`) | Identifies the customer domain |
| `X-Push-Invalidation` | `enabled` | Enables push-based cache purging to your CDN |

**Required CDN behavior:**

- Enable gzip compression
- Forward query parameters to origin and include them in cache keys
- Suppress or reset the `Age` response header to zero
- Respect origin `Cache-Control` headers for TTL

**Correct -- Cloudflare Worker origin configuration:**

```javascript
// Cloudflare Worker for EDS BYO CDN
export default {
  async fetch(request) {
    const url = new URL(request.url);
    const origin = 'https://main--mysite--myorg.aem.live';

    const originRequest = new Request(origin + url.pathname + url.search, {
      method: request.method,
      headers: {
        ...Object.fromEntries(request.headers),
        'X-Forwarded-Host': 'www.example.com',
        'X-Push-Invalidation': 'enabled',
      },
    });

    return fetch(originRequest);
  },
};
```

**Incorrect -- missing required headers:**

```javascript
// WRONG: Missing X-Forwarded-Host and X-Push-Invalidation
const originRequest = new Request(origin + url.pathname, {
  method: request.method,
});
// Results: wrong content served, no automatic cache purging
```

Always validate your BYO CDN configuration in a staging environment before going live.

---

### 3. Caching Behavior

#### 3.1 Default Cache TTLs

**Preview tier (.aem.page):**

| Content Type | Cache-Control |
|-------------|---------------|
| HTML, JSON, code | `max-age=60, must-revalidate` |
| Media (images, video) | `max-age=2592000` (30 days) |
| Media errors (3xx-5xx) | `max-age=3600` (1 hour) |

**Live tier (.aem.live) and production:**

| Content Type | Cache-Control |
|-------------|---------------|
| HTML, JSON, code | `max-age=7200, must-revalidate` (2 hours) |
| Media (images, video) | `max-age=2592000` (30 days) |
| Media errors (3xx-5xx) | `max-age=3600` (1 hour) |

#### 3.2 CDN Cache vs Browser Cache

The `must-revalidate` directive means browsers must check with the CDN when the `max-age` expires, but the CDN may still serve from its cache if the content has not changed (304 Not Modified).

For **media assets**, the content-hash URL scheme means assets are immutable. When an author replaces an image, a new URL is generated, so the old cached version is never stale -- it simply stops being referenced.

#### 3.3 Push-Based Invalidation

Unlike traditional CDN configurations that rely on TTL expiration, EDS uses **push invalidation**:

```
Author publishes page via Sidekick
  --> EDS pipeline processes content
  --> CDN cache purged by URL and cache tag/key
  --> Next request fetches fresh content from origin
```

This means:
- Content updates appear immediately after publish (no TTL wait)
- No manual purge actions needed for content changes
- Works with both Adobe-managed CDN and BYO CDN (when `X-Push-Invalidation: enabled` is set)

#### 3.4 Vary Header

EDS sets `Vary: Accept-Encoding, X-Forwarded-Host` on responses. This means:
- Different encodings (gzip, brotli) are cached separately
- Different `X-Forwarded-Host` values create separate cache entries (important for multi-domain sites)

---

### 4. CDN Headers and Configuration

Custom HTTP response headers are configured in your site's configuration (fstab / site config), not in code.

#### 4.1 Custom Headers Configuration

Headers are defined in a `headers` configuration object using glob URL patterns:

```json
{
  "headers": {
    "/api/**": [
      {
        "key": "Access-Control-Allow-Origin",
        "value": "https://www.example.com"
      },
      {
        "key": "Access-Control-Allow-Methods",
        "value": "GET, POST, OPTIONS"
      }
    ],
    "/**": [
      {
        "key": "X-Frame-Options",
        "value": "SAMEORIGIN"
      },
      {
        "key": "X-Content-Type-Options",
        "value": "nosniff"
      },
      {
        "key": "Strict-Transport-Security",
        "value": "max-age=31536000; includeSubDomains"
      },
      {
        "key": "Permissions-Policy",
        "value": "camera=(), microphone=(), geolocation=()"
      }
    ]
  }
}
```

**URL pattern matching:**
- `*` -- single level wildcard (prefix or suffix)
- `**` -- deep path matching across directory levels
- `/foo/**` -- matches `/foo/bar`, `/foo/bar/baz`, etc.

#### 4.2 CORS Configuration

**Correct -- restrict to specific origin:**

```json
{
  "headers": {
    "/fragments/**": [
      {
        "key": "Access-Control-Allow-Origin",
        "value": "https://www.example.com"
      }
    ]
  }
}
```

**Incorrect -- wildcard CORS on all paths:**

```json
{
  "headers": {
    "/**": [
      {
        "key": "Access-Control-Allow-Origin",
        "value": "*"
      }
    ]
  }
}
```

A wildcard `*` value for `Access-Control-Allow-Origin` renders preview and live environments vulnerable to CSRF attacks against authors logged in via AEM Sidekick. Always restrict to specific origins and specific URL paths.

#### 4.3 Content Security Policy (CSP)

EDS v5+ includes CSP with `strict-dynamic` and cached nonce by default. The system automatically replaces the `aem` placeholder in both the policy and script attributes with a cryptographic random nonce string for every request.

```html
<!-- EDS automatically generates nonce-based CSP -->
<meta http-equiv="Content-Security-Policy"
      content="script-src 'nonce-abc123' 'strict-dynamic'; ...">
<script nonce="abc123" src="/scripts/scripts.js"></script>
```

The nonce changes per request, preventing XSS attacks while allowing legitimate scripts.

#### 4.4 HSTS

EDS enforces HTTPS with HTTP Strict Transport Security headers by default. The `Strict-Transport-Security` header prevents protocol downgrade attacks.

#### 4.5 X-Robots-Tag

EDS automatically sets `X-Robots-Tag: noindex, nofollow` on preview and live origins (`.aem.page` and `.aem.live`). The CDN removes this header when serving through a production custom domain, ensuring only your branded domain is indexed by search engines.

---

### 5. Redirects

EDS manages redirects through a spreadsheet in your content source, processed at the CDN edge for fast resolution.

#### 5.1 Redirect Spreadsheet

Create a spreadsheet named `redirects` (or `redirects.xlsx`) in your project root folder with two required columns:

| Source | Destination |
|--------|-------------|
| `/old-page` | `/new-page` |
| `/legacy/product` | `https://shop.example.com/product` |
| `/about-us` | `/company/about` |

**Column definitions:**

- **Source**: Relative path on your domain (e.g., `/old-page`)
- **Destination**: Relative path for internal redirects or fully qualified URL for external redirects

**Important constraints:**

- EDS spreadsheet redirects only support **301 (Moved Permanently)** status codes -- this is by design for optimal caching
- Other redirect types (302, 307, 308) must be configured at the CDN level
- If your workbook has multiple sheets, redirects only work on the sheet named `helix-default`
- Redirects take precedence over existing content at the same URL
- Removing a redirect restores access to the underlying page (if published)

#### 5.2 Wildcard Redirects

Pattern-based redirects require CDN-level configuration (not the spreadsheet):

```
/old-section/*  -->  /new-section/$1
```

Use wildcards sparingly. The documentation warns that wildcard redirects have "a set of issues around unmanaged complexity" and may create redirects that resolve to 404 errors.

#### 5.3 Redirect Best Practices for Site Migration

1. Map high-traffic URLs individually via the redirect spreadsheet
2. Use wildcard CDN rules only for clearly structured path changes
3. Avoid bulk redirects to the homepage (use specific destinations)
4. Update your sitemap after migration
5. Monitor 404 errors post-launch and add redirects as needed
6. Preview redirect changes via Sidekick before publishing to production

#### 5.4 Redirect Workflow

```
1. Edit redirects spreadsheet in SharePoint/Google Drive/AEM Author
2. Preview via Sidekick (validates on .aem.page)
3. Publish via Sidekick (applies to .aem.live and custom domain)
4. CDN cache purged automatically (push invalidation)
5. Redirects active at CDN edge within seconds
```

---

### 6. Edge Compute and Advanced CDN Features

#### 6.1 BYO CDN Edge Functions

When using a BYO CDN, you can implement edge logic on your CDN layer:

**A/B testing at CDN level:**

```javascript
// Cloudflare Worker example: A/B test routing
export default {
  async fetch(request) {
    const url = new URL(request.url);

    // Assign variant based on cookie or random
    let variant = getCookie(request, 'ab-variant');
    if (!variant) {
      variant = Math.random() < 0.5 ? 'control' : 'experiment';
    }

    // Route to different EDS branch per variant
    const origin = variant === 'experiment'
      ? 'https://experiment--mysite--myorg.aem.live'
      : 'https://main--mysite--myorg.aem.live';

    const response = await fetch(origin + url.pathname + url.search, {
      headers: {
        'X-Forwarded-Host': 'www.example.com',
        'X-Push-Invalidation': 'enabled',
      },
    });

    // Set variant cookie for consistency
    const newResponse = new Response(response.body, response);
    newResponse.headers.set('Set-Cookie',
      `ab-variant=${variant}; Path=/; Max-Age=86400; SameSite=Lax`);
    return newResponse;
  },
};
```

This pattern uses EDS branch-based environments to serve different content variants from separate branches, letting the CDN edge decide which branch to fetch.

#### 6.2 Geo-Based Routing

```javascript
// Route to localized content based on country
const countryMap = {
  DE: '/de/',
  FR: '/fr/',
  JP: '/ja/',
};

const country = request.headers.get('CF-IPCountry'); // Cloudflare-specific
const prefix = countryMap[country] || '/en/';
```

#### 6.3 Authentication at Edge

For protecting staging or pre-launch sites, use your CDN's authentication features:

- **Cloudflare:** Zero Trust Access policies
- **Akamai:** Edge authentication rules
- **Fastly:** Basic auth via VCL
- **CloudFront:** Lambda@Edge with signed cookies

EDS does not provide built-in edge authentication. Use your BYO CDN or a third-party service.

#### 6.4 Rate Limiting

Rate limiting is not built into the EDS CDN. Implement it at your BYO CDN layer:

- Cloudflare: Rate Limiting Rules
- Akamai: Rate Controls
- Fastly: Rate Limiting via VCL
- CloudFront: AWS WAF rate-based rules

---

### 7. Performance Tuning

#### 7.1 Push Invalidation = Instant Purge

The single most important EDS CDN advantage: publish triggers immediate cache purge. No stale content window. No TTL tuning required for freshness.

This means you can safely use long CDN TTLs (hours) because the cache will be proactively purged whenever content changes. The TTL only matters for the rare case where a CDN node has not yet received the purge signal.

#### 7.2 Preconnect Hints

For custom domains with BYO CDN, add preconnect hints for your origin domain in `head.html`:

```html
<link rel="preconnect" href="https://www.example.com" crossorigin>
```

#### 7.3 Protocol Support

| Feature | Status |
|---------|--------|
| HTTP/1.1 | Supported |
| HTTP/2 | Supported |
| TLS 1.2 | Required minimum |
| IPv6 | Not supported |

DNS TTL for the delivery stack: 10 minutes (automatic switching).

#### 7.4 Compression

EDS origin serves content with standard compression. When using BYO CDN, ensure gzip compression is enabled on your CDN. The CDN should accept compressed content from origin and potentially re-compress for clients.

#### 7.5 Image Optimization

Media assets in EDS use content-hash URLs enabling permanent caching. The EDS pipeline handles image optimization at the content processing stage:

- Automatic format selection (WebP/AVIF where supported)
- Responsive `<picture>` / `srcset` generation in blocks
- Lazy loading for below-fold images (automatic)
- Eager loading for LCP image (automatic)

The CDN serves pre-optimized media from the Media Bus -- no CDN-level image transformation is needed.

#### 7.6 Monitoring with RUM Dashboard

EDS includes Real User Monitoring (RUM) built in (referred to as "Operational Telemetry"):

- **Core Web Vitals**: LCP, CLS, INP measured from real users
- **Page load performance**: TTFB, FCP, onload timing
- **Traffic analytics**: page views, unique visitors, top pages
- **Error tracking**: 404 rates, JavaScript errors
- **Device/browser breakdown**: performance by segment

Access RUM data through the EDS dashboard or integrate with Google PageSpeed Insights for automated checks.

```
https://www.aem.live/tools/rum/<your-domain>
```

---

### Quick Reference: EDS CDN Configuration Checklist

1. **Custom domain**: CNAME `www` to `cdn.adobeaemcloud.com`, set A records for apex
2. **Push invalidation**: Add `cdn.prod.host` and `cdn.prod.type: managed` to `.helix/config`
3. **Redirects**: Create `redirects` spreadsheet in project root with Source/Destination columns
4. **Security headers**: Configure in headers config object with glob URL patterns
5. **CORS**: Restrict `Access-Control-Allow-Origin` to specific domains (never `*`)
6. **BYO CDN**: Set `X-Forwarded-Host` and `X-Push-Invalidation: enabled` headers to origin
7. **Monitor**: Check RUM dashboard for Core Web Vitals and cache performance
8. **Media**: Let content-hash URLs handle immutable caching -- no manual cache busting needed
