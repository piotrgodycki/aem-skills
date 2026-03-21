---
title: Edge Compute and Edge Workers for EDS
impact: HIGH
impactDescription: Edge compute enables server-side logic at CDN edge — incorrect patterns bypass EDS caching, add latency, and create debugging nightmares
tags: eds, edge-compute, edge-workers, cloudflare-workers, cdn, byocdn, personalization, geolocation, authentication, a-b-testing
---

## Edge Compute and Edge Workers for EDS

Edge compute runs server-side logic at CDN points of presence, close to users. With EDS's BYOCDN capability, you can use Cloudflare Workers, Akamai EdgeWorkers, or AWS CloudFront Functions to add authentication, personalization, and API proxying without client-side JavaScript.

---

### 1. Architecture

```
User Request
  │
  ▼
BYOCDN Edge (Cloudflare / Akamai / CloudFront)
  ├── Edge Worker (your code runs here)
  │   ├── Authentication check
  │   ├── Geolocation routing
  │   ├── A/B test assignment
  │   ├── Header manipulation
  │   └── API proxy
  │
  ▼
Adobe EDS CDN (origin)
  ├── Push invalidation (content freshness)
  └── Static content delivery
  │
  ▼
AEM Author / Content Source
```

**Key principle:** Edge workers sit between the user and EDS CDN. They can modify requests going to origin and responses going to users, but should not replace EDS's content delivery.

---

### 2. BYOCDN Setup with Edge Workers

#### Cloudflare Workers

```javascript
// wrangler.toml
// name = "eds-edge-worker"
// main = "src/index.js"
// compatibility_date = "2024-01-01"

// [vars]
// EDS_ORIGIN = "main--mysite--myorg.aem.live"

export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // Route to EDS origin
    const originUrl = new URL(url.pathname + url.search, `https://${env.EDS_ORIGIN}`);
    const originRequest = new Request(originUrl, {
      method: request.method,
      headers: request.headers,
    });

    const response = await fetch(originRequest);

    // Modify response headers
    const newResponse = new Response(response.body, response);
    newResponse.headers.set('X-Edge-Location', request.cf?.colo || 'unknown');

    return newResponse;
  },
};
```

#### Akamai EdgeWorkers

```javascript
// main.js — Akamai EdgeWorker
import { httpRequest } from 'http-request';
import { createResponse } from 'create-response';

export async function responseProvider(request) {
  const originResponse = await httpRequest(`https://${request.host}${request.url}`);
  return createResponse(originResponse.status, originResponse.getHeaders(), originResponse.body);
}

export function onClientRequest(request) {
  // Modify request before it reaches origin
  request.setHeader('X-Geo-Country', request.userLocation.country);
}
```

---

### 3. A/B Testing at the Edge

Eliminate client-side flicker by assigning variants at the edge:

```javascript
// Cloudflare Worker — edge-side A/B testing
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // Only A/B test specific pages
    if (!url.pathname.startsWith('/landing/')) {
      return fetch(request);
    }

    // Get or assign variant from cookie
    const cookies = parseCookies(request.headers.get('Cookie') || '');
    let variant = cookies['ab-variant'];

    if (!variant) {
      variant = Math.random() < 0.5 ? 'control' : 'challenger';
    }

    // Fetch the appropriate variant
    const originUrl = variant === 'challenger'
      ? `https://${env.EDS_ORIGIN}${url.pathname}?variant=challenger`
      : `https://${env.EDS_ORIGIN}${url.pathname}`;

    const response = await fetch(originUrl, {
      headers: { ...Object.fromEntries(request.headers) },
    });

    const newResponse = new Response(response.body, response);

    // Set variant cookie (persists assignment)
    if (!cookies['ab-variant']) {
      newResponse.headers.append('Set-Cookie',
        `ab-variant=${variant}; Path=/; Max-Age=2592000; SameSite=Lax`);
    }

    // Add variant header for RUM tracking
    newResponse.headers.set('X-AB-Variant', variant);

    return newResponse;
  },
};

function parseCookies(cookieStr) {
  return Object.fromEntries(
    cookieStr.split(';').map((c) => c.trim().split('=').map((v) => v.trim()))
  );
}
```

---

### 4. Geolocation-Based Content

```javascript
// Serve region-specific content based on user location
export default {
  async fetch(request, env) {
    const country = request.cf?.country || 'US';
    const url = new URL(request.url);

    // Redirect to localized content
    const regionMap = {
      DE: '/de/', AT: '/de/', CH: '/de/',
      FR: '/fr/', BE: '/fr/',
      ES: '/es/', MX: '/es/',
      JP: '/ja/',
    };

    // Only redirect root or unlocalized paths
    if (url.pathname === '/' && regionMap[country]) {
      return Response.redirect(`${url.origin}${regionMap[country]}`, 302);
    }

    // Pass geo headers to origin for content personalization
    const modifiedRequest = new Request(request);
    modifiedRequest.headers.set('X-Geo-Country', country);
    modifiedRequest.headers.set('X-Geo-City', request.cf?.city || '');
    modifiedRequest.headers.set('X-Geo-Region', request.cf?.region || '');

    return fetch(modifiedRequest);
  },
};
```

---

### 5. Authentication at the Edge

```javascript
// Edge authentication — protect content before it reaches the user
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // Public paths — no auth needed
    const publicPaths = ['/', '/blog/', '/about', '/login'];
    if (publicPaths.some((p) => url.pathname.startsWith(p))) {
      return fetch(request);
    }

    // Protected paths — require valid session
    const sessionToken = getCookie(request, 'session');

    if (!sessionToken) {
      return Response.redirect(`${url.origin}/login?redirect=${encodeURIComponent(url.pathname)}`, 302);
    }

    // Validate session against auth service
    const isValid = await validateSession(sessionToken, env);
    if (!isValid) {
      return Response.redirect(`${url.origin}/login?expired=true`, 302);
    }

    // Forward to origin with user context
    const modifiedRequest = new Request(request);
    modifiedRequest.headers.set('X-User-Authenticated', 'true');

    return fetch(modifiedRequest);
  },
};

async function validateSession(token, env) {
  try {
    const response = await fetch(`${env.AUTH_SERVICE_URL}/validate`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ token }),
    });
    return response.ok;
  } catch {
    return false;
  }
}

function getCookie(request, name) {
  const cookies = request.headers.get('Cookie') || '';
  const match = cookies.match(new RegExp(`${name}=([^;]+)`));
  return match ? match[1] : null;
}
```

---

### 6. API Proxy

Proxy third-party APIs to avoid CORS issues and hide API keys:

```javascript
// Proxy requests to /api/* to backend services
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    if (url.pathname.startsWith('/api/')) {
      return handleApi(request, url, env);
    }

    // Non-API requests go to EDS origin
    return fetch(request);
  },
};

async function handleApi(request, url, env) {
  const path = url.pathname.replace('/api/', '');

  const apiMap = {
    'products': env.PRODUCT_API_URL,
    'search': env.SEARCH_API_URL,
    'newsletter': env.NEWSLETTER_API_URL,
  };

  const segment = path.split('/')[0];
  const targetBase = apiMap[segment];

  if (!targetBase) {
    return new Response('Not Found', { status: 404 });
  }

  const targetUrl = `${targetBase}/${path.replace(`${segment}/`, '')}${url.search}`;

  const response = await fetch(targetUrl, {
    method: request.method,
    headers: {
      'Content-Type': request.headers.get('Content-Type') || 'application/json',
      'Authorization': `Bearer ${env.API_KEY}`, // API key stays at edge, not exposed to client
    },
    body: request.method !== 'GET' ? await request.text() : undefined,
  });

  const newResponse = new Response(response.body, response);
  newResponse.headers.set('Access-Control-Allow-Origin', url.origin);
  newResponse.headers.set('Cache-Control', 'public, max-age=60');

  return newResponse;
}
```

---

### 7. Header Manipulation

```javascript
// Add security and performance headers
export default {
  async fetch(request, env) {
    const response = await fetch(request);
    const newResponse = new Response(response.body, response);

    // Security headers
    newResponse.headers.set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    newResponse.headers.set('X-Content-Type-Options', 'nosniff');
    newResponse.headers.set('X-Frame-Options', 'SAMEORIGIN');
    newResponse.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
    newResponse.headers.set('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');

    // CSP header
    newResponse.headers.set('Content-Security-Policy',
      "default-src 'self'; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com; " +
      "style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; " +
      "connect-src 'self' https://rum.hlx.page https://www.google-analytics.com");

    return newResponse;
  },
};
```

---

### 8. Anti-Patterns

#### Heavy Computation at the Edge

```javascript
// WRONG — complex processing at edge adds latency
async fetch(request) {
  const html = await (await fetch(request)).text();
  const dom = new HTMLRewriter().on('*', transformAllElements).transform(html);
  // DOM parsing + transformation at edge = slow

  // CORRECT — do heavy processing in origin or client
  // Edge should only do lightweight request/response transforms
}
```

#### Bypassing EDS Caching

```javascript
// WRONG — adding no-cache to all responses
newResponse.headers.set('Cache-Control', 'no-store');
// Defeats the entire EDS CDN caching layer

// CORRECT — respect EDS cache headers, only modify when necessary
// Let EDS push invalidation handle freshness
```

#### Storing State at the Edge

```javascript
// WRONG — using global variables for state (workers are stateless)
let requestCount = 0;
export default {
  async fetch(request) {
    requestCount++; // Unreliable — multiple isolates, no shared state
  },
};

// CORRECT — use KV, Durable Objects, or external storage for state
const count = await env.COUNTER_KV.get('requests');
```

#### Not Handling Edge Worker Errors

```javascript
// WRONG — edge error = user sees error page
export default {
  async fetch(request, env) {
    const user = await validateUser(request); // throws on auth service down
    // If auth service is down, entire site is down!
  },
};

// CORRECT — fail open or provide fallback
export default {
  async fetch(request, env) {
    try {
      const user = await validateUser(request);
    } catch {
      // Auth service down — serve public content, log error
      return fetch(request); // fail open to EDS origin
    }
  },
};
```
