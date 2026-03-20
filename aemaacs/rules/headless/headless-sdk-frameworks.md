---
title: AEM Headless SDK & Framework Integration
impact: HIGH
impactDescription: Proper SDK usage and framework integration patterns determine performance, cacheability, and reliability of headless frontends
tags: headless, sdk, react, nextjs, vue, svelte, authentication, cors, images, caching
---

## AEM Headless SDK & Framework Integration

AEM provides official SDKs for JavaScript, React, and Next.js to consume Content Fragment data via persisted GraphQL queries. External frontends must handle authentication, CORS, image delivery, and multi-environment configuration correctly.

### JavaScript SDK — @adobe/aem-headless-client-js

```bash
npm install @adobe/aem-headless-client-js
```

#### Initialization

```javascript
import AEMHeadless from '@adobe/aem-headless-client-js';

// Publish tier — no auth required for public content
const aemHeadlessClient = new AEMHeadless({
  serviceURL: process.env.AEM_HOST,               // https://publish-p123-e456.adobeaemcloud.com
  endpoint: '/content/cq:graphql/wknd-shared/endpoint.json',
});

// Author tier — requires authentication
const aemAuthorClient = new AEMHeadless({
  serviceURL: process.env.AEM_AUTHOR_HOST,
  endpoint: '/content/cq:graphql/wknd-shared/endpoint.json',
  auth: process.env.AEM_AUTH_TOKEN,                // Bearer token or dev token
});
```

#### Running Persisted Queries

```javascript
// Correct: persisted queries execute via HTTP GET — CDN-cacheable
async function fetchPersistedQuery(persistedQueryName, queryParameters) {
  let data;
  let errors;

  try {
    const response = await aemHeadlessClient.runPersistedQuery(
      persistedQueryName,
      queryParameters
    );
    data = response?.data;
  } catch (e) {
    console.error(e.toJSON());
    errors = e;
  }

  return { data, errors };
}

// Usage
const { data, errors } = await fetchPersistedQuery(
  'wknd-shared/adventures-by-slug',
  { slug: 'bali-surf-camp' }
);
```

#### Webpack 5+ Polyfill Dependencies

```json
{
  "devDependencies": {
    "buffer": "npm:buffer@^6.0.3",
    "util": "npm:util@^0.12.5"
  }
}
```

### React SDK — @adobe/aem-headless-client-react

```bash
npm install @adobe/aem-headless-client-js @adobe/aem-headless-client-react
```

#### Context Provider Setup

```jsx
import { AEMHeadlessProvider } from '@adobe/aem-headless-client-react';

function App() {
  return (
    <AEMHeadlessProvider
      host={process.env.REACT_APP_AEM_HOST}
      endpoint="/content/cq:graphql/wknd-shared/endpoint.json"
      token={process.env.REACT_APP_DEV_TOKEN}
    >
      <AdventuresList />
    </AEMHeadlessProvider>
  );
}
```

#### Custom Hooks Pattern

```javascript
import { useEffect, useState } from 'react';
import AEMHeadless from '@adobe/aem-headless-client-js';

const aemHeadlessClient = new AEMHeadless({
  serviceURL: process.env.REACT_APP_AEM_HOST,
  endpoint: '/content/cq:graphql/wknd-shared/endpoint.json',
});

// Hook: fetch all adventures
export function useAdventuresByActivity(activity) {
  const [adventures, setAdventures] = useState(null);
  const [errors, setErrors] = useState(null);

  useEffect(() => {
    async function fetchData() {
      const queryName = activity
        ? 'wknd-shared/adventures-by-activity'
        : 'wknd-shared/adventures-all';
      const params = activity ? { activity } : {};

      const { data, err } = await fetchPersistedQuery(queryName, params);

      if (err) {
        setErrors(err);
      } else {
        setAdventures(data?.adventureList?.items || []);
      }
    }
    fetchData();
  }, [activity]);

  return { adventures, errors };
}

// Hook: fetch single adventure by slug
export function useAdventureBySlug(slug) {
  const [adventure, setAdventure] = useState(null);
  const [errors, setErrors] = useState(null);

  useEffect(() => {
    async function fetchData() {
      const { data, err } = await fetchPersistedQuery(
        'wknd-shared/adventure-by-slug',
        { slug }
      );

      if (err) {
        setErrors(err);
      } else if (data?.adventureList?.items?.length === 1) {
        setAdventure(data.adventureList.items[0]);
      }
    }
    fetchData();
  }, [slug]);

  return { adventure, errors };
}
```

#### React Environment Variables

```bash
# .env.development
REACT_APP_AEM_HOST=https://publish-p123-e456.adobeaemcloud.com
REACT_APP_AUTH_METHOD=dev-token
REACT_APP_DEV_TOKEN=<local-development-token>
REACT_APP_GRAPHQL_ENDPOINT=/content/cq:graphql/wknd-shared/endpoint.json

# .env.production
REACT_APP_AEM_HOST=https://publish-p123-e456.adobeaemcloud.com
REACT_APP_AUTH_METHOD=
```

#### Development Proxy (src/setupProxy.js)

```javascript
const { createProxyMiddleware } = require('http-proxy-middleware');

module.exports = function (app) {
  app.use(
    ['/content', '/graphql'],
    createProxyMiddleware({
      target: process.env.REACT_APP_AEM_HOST,
      changeOrigin: true,
      headers: {
        Authorization: `Bearer ${process.env.REACT_APP_DEV_TOKEN}`,
      },
    })
  );
};
```

### Next.js Integration — @adobe/aem-headless-client-nextjs

```bash
npm install @adobe/aem-headless-client-js
```

#### Pages Router: getStaticProps with ISR

```javascript
// lib/aem-headless.js
import AEMHeadless from '@adobe/aem-headless-client-js';

export const aemClient = new AEMHeadless({
  serviceURL: process.env.AEM_HOST,
  endpoint: '/content/cq:graphql/wknd-shared/endpoint.json',
});

// pages/adventures/[slug].js
export async function getStaticProps({ params, preview }) {
  const host = preview ? process.env.AEM_AUTHOR_HOST : process.env.AEM_HOST;
  const client = new AEMHeadless({
    serviceURL: host,
    endpoint: '/content/cq:graphql/wknd-shared/endpoint.json',
    auth: preview ? process.env.AEM_PREVIEW_TOKEN : undefined,
  });

  const { data } = await client.runPersistedQuery(
    'wknd-shared/adventure-by-slug',
    { slug: params.slug }
  );

  return {
    props: { adventure: data?.adventureBySlug?.item || null },
    revalidate: 60, // ISR: regenerate every 60 seconds
  };
}

export async function getStaticPaths() {
  const { data } = await aemClient.runPersistedQuery('wknd-shared/adventures-all');
  const paths = data?.adventureList?.items?.map((item) => ({
    params: { slug: item.slug },
  })) || [];

  return { paths, fallback: 'blocking' };
}
```

#### App Router: React Server Components

```javascript
// app/adventures/[slug]/page.js
import AEMHeadless from '@adobe/aem-headless-client-js';

const aemClient = new AEMHeadless({
  serviceURL: process.env.AEM_HOST,
  endpoint: '/content/cq:graphql/wknd-shared/endpoint.json',
});

// Server Component — fetches data at request time or build time
export default async function AdventurePage({ params }) {
  const { data } = await aemClient.runPersistedQuery(
    'wknd-shared/adventure-by-slug',
    { slug: params.slug }
  );
  const adventure = data?.adventureBySlug?.item;

  if (!adventure) return <div>Adventure not found</div>;

  return (
    <article>
      <h1>{adventure.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: adventure.description?.html }} />
    </article>
  );
}

// Static generation with revalidation
export const revalidate = 60;

export async function generateStaticParams() {
  const { data } = await aemClient.runPersistedQuery('wknd-shared/adventures-all');
  return data?.adventureList?.items?.map((item) => ({
    slug: item.slug,
  })) || [];
}
```

#### Next.js Environment Variables

```bash
# .env.local
AEM_HOST=https://publish-p123-e456.adobeaemcloud.com
AEM_AUTHOR_HOST=https://author-p123-e456.adobeaemcloud.com
AEM_PREVIEW_TOKEN=<dev-token-for-preview>

# Public vars (exposed to browser)
NEXT_PUBLIC_AEM_HOST=https://publish-p123-e456.adobeaemcloud.com
```

### Vue / Nuxt with AEM Headless

```javascript
// composables/useAemHeadless.js
import AEMHeadless from '@adobe/aem-headless-client-js';

const aemClient = new AEMHeadless({
  serviceURL: import.meta.env.VITE_AEM_HOST,
  endpoint: '/content/cq:graphql/wknd-shared/endpoint.json',
});

export function useAdventures() {
  const adventures = ref([]);
  const error = ref(null);

  async function fetchAdventures(activity) {
    try {
      const { data } = await aemClient.runPersistedQuery(
        activity
          ? 'wknd-shared/adventures-by-activity'
          : 'wknd-shared/adventures-all',
        activity ? { activity } : {}
      );
      adventures.value = data?.adventureList?.items || [];
    } catch (e) {
      error.value = e;
    }
  }

  return { adventures, error, fetchAdventures };
}
```

```javascript
// Nuxt: server/api/adventures.js (server route)
import AEMHeadless from '@adobe/aem-headless-client-js';

export default defineEventHandler(async (event) => {
  const client = new AEMHeadless({
    serviceURL: process.env.AEM_HOST,
    endpoint: '/content/cq:graphql/wknd-shared/endpoint.json',
  });
  const { data } = await client.runPersistedQuery('wknd-shared/adventures-all');
  return data?.adventureList?.items || [];
});
```

### SvelteKit with AEM Headless

```javascript
// src/lib/aem.js
import AEMHeadless from '@adobe/aem-headless-client-js';

export const aemClient = new AEMHeadless({
  serviceURL: import.meta.env.VITE_AEM_HOST,
  endpoint: '/content/cq:graphql/wknd-shared/endpoint.json',
});

// src/routes/adventures/[slug]/+page.server.js
import { aemClient } from '$lib/aem';

export async function load({ params }) {
  const { data } = await aemClient.runPersistedQuery(
    'wknd-shared/adventure-by-slug',
    { slug: params.slug }
  );
  return { adventure: data?.adventureBySlug?.item };
}
```

### Authentication Patterns

AEM supports three authentication modes depending on the environment.

#### Local Development Token

Short-lived token generated from AEM Developer Console. For development only.

```javascript
const client = new AEMHeadless({
  serviceURL: 'https://author-p123-e456.adobeaemcloud.com',
  endpoint: '/content/cq:graphql/wknd-shared/endpoint.json',
  auth: 'eyJhbGci...dev-token',  // Expires in 24 hours
});
```

#### Service Credentials (Server-to-Server)

Production-grade. The external application exchanges a JWT for an access token.

```javascript
import auth from '@adobe/jwt-auth';

async function getAccessToken(serviceCredentials) {
  const { access_token } = await auth({
    clientId: serviceCredentials.technicalAccount.clientId,
    technicalAccountId: serviceCredentials.id,
    orgId: serviceCredentials.org,
    clientSecret: serviceCredentials.technicalAccount.clientSecret,
    privateKey: serviceCredentials.privateKey,
    metaScopes: serviceCredentials.metascopes.split(','),
    ims: `https://${serviceCredentials.imsEndpoint}`,
  });
  return access_token;
}

// Use in AEM Headless client
const token = await getAccessToken(credentials.integration);
const client = new AEMHeadless({
  serviceURL: process.env.AEM_AUTHOR_HOST,
  endpoint: '/content/cq:graphql/wknd-shared/endpoint.json',
  auth: token,
});
```

#### Bearer Token in HTTP Headers

```javascript
const response = await fetch(
  `${process.env.AEM_HOST}/graphql/execute.json/wknd-shared/adventures-all`,
  {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${accessToken}`,
    },
  }
);
```

**Incorrect — hardcoded credentials:**

```javascript
// Never hardcode credentials or tokens
const client = new AEMHeadless({
  serviceURL: 'https://author-p123-e456.adobeaemcloud.com',
  auth: 'eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.hardcoded-token-value',
});
```

**Correct — environment variables:**

```javascript
const client = new AEMHeadless({
  serviceURL: process.env.AEM_HOST,
  auth: process.env.AEM_AUTH_TOKEN,
});
```

### CORS Configuration

Browser-based headless clients on a different origin require CORS configuration on AEM. Create an OSGi config file deployed via Cloud Manager.

#### Config File

Path: `ui.config/src/main/content/jcr_root/apps/<project>/osgiconfig/config.publish/com.adobe.granite.cors.impl.CORSPolicyImpl~headless.cfg.json`

```json
{
  "supportscredentials": false,
  "supportedmethods": ["GET", "HEAD", "POST"],
  "exposedheaders": [""],
  "alloworigin": [
    "https://my-frontend.example.com",
    "https://staging.example.com"
  ],
  "maxage:Integer": 1800,
  "alloworiginregexp": ["https://deploy-preview-\\d+\\.example\\.com"],
  "supportedheaders": [
    "Origin",
    "Accept",
    "X-Requested-With",
    "Content-Type",
    "Access-Control-Request-Method",
    "Access-Control-Request-Headers",
    "Authorization"
  ],
  "allowedpaths": [
    "/content/cq:graphql/wknd-shared/endpoint.json",
    "/graphql/execute.json/.*"
  ]
}
```

**Incorrect — wildcard origin in production:**

```json
{
  "alloworigin": ["*"],
  "allowedpaths": ["/.*"]
}
```

**Correct — explicit origins and paths:**

```json
{
  "alloworigin": ["https://my-frontend.example.com"],
  "allowedpaths": ["/graphql/execute.json/.*"]
}
```

### Image Delivery — _dynamicUrl

`_dynamicUrl` returns a relative path for web-optimized image delivery. The frontend must prepend the AEM host.

#### GraphQL Query

```graphql
query ($path: String!, $imageWidth: Int, $imageFormat: AssetTransformFormat = JPG) {
  adventureByPath(
    _path: $path
    _assetTransform: {
      format: $imageFormat
      width: $imageWidth
      quality: 80
      preferWebp: true
    }
  ) {
    item {
      title
      primaryImage {
        ... on ImageRef {
          _dynamicUrl
        }
      }
    }
  }
}
```

#### React Responsive Image Component

```jsx
const AEM_HOST = process.env.REACT_APP_AEM_HOST;

function AdventureImage({ dynamicUrl, alt }) {
  // _dynamicUrl does NOT include the AEM domain — prepend it
  const imageUrl = `${AEM_HOST}${dynamicUrl}`;

  return (
    <picture>
      <source
        srcSet={`${imageUrl}&width=2600`}
        media="(min-width: 2001px)"
      />
      <source
        srcSet={`${imageUrl}&width=2000`}
        media="(min-width: 1000px)"
      />
      <img
        src={`${imageUrl}&width=800`}
        srcSet={`
          ${imageUrl}&width=800 800w,
          ${imageUrl}&width=1600 1600w,
          ${imageUrl}&width=2000 2000w
        `}
        sizes="calc(100vw - 10rem)"
        alt={alt}
        loading="lazy"
      />
    </picture>
  );
}
```

**Incorrect — using _dynamicUrl directly without host:**

```jsx
// _dynamicUrl is a relative path — this resolves against the frontend domain
<img src={data.adventureByPath.item.primaryImage._dynamicUrl} />
```

**Correct — prepend AEM host:**

```jsx
<img src={`${process.env.REACT_APP_AEM_HOST}${data.adventureByPath.item.primaryImage._dynamicUrl}`} />
```

#### Supported _assetTransform Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `format` | `AssetTransformFormat` | `JPG`, `PNG`, `WEBP`, `GIF` |
| `width` | `Int` | Desired width in pixels |
| `quality` | `Int` | Compression quality (1-100) |
| `preferWebp` | `Boolean` | Serve WebP when browser supports it |
| `crop` | `String` | Crop coordinates: `"x,y,width,height"` |
| `size` | `Object` | `{ width, height }` |
| `rotation` | `Int` | Degrees of rotation |
| `flip` | `String` | `HORIZONTAL`, `VERTICAL` |

### Error Handling and Retry

```javascript
async function fetchWithRetry(queryName, params, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await aemHeadlessClient.runPersistedQuery(queryName, params);
      return { data: response?.data, errors: null };
    } catch (error) {
      const status = error?.statusCode;

      // Do not retry client errors (4xx) except 429
      if (status && status >= 400 && status < 500 && status !== 429) {
        return { data: null, errors: error.toJSON() };
      }

      if (attempt === maxRetries) {
        return { data: null, errors: error.toJSON() };
      }

      // Exponential backoff
      const delay = Math.min(1000 * Math.pow(2, attempt - 1), 10000);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
}
```

### Caching Strategies

```javascript
// In-memory cache for client-side apps
const queryCache = new Map();

async function cachedQuery(queryName, params, ttlMs = 60000) {
  const cacheKey = `${queryName}:${JSON.stringify(params)}`;
  const cached = queryCache.get(cacheKey);

  if (cached && Date.now() - cached.timestamp < ttlMs) {
    return cached.data;
  }

  const { data } = await fetchPersistedQuery(queryName, params);
  queryCache.set(cacheKey, { data, timestamp: Date.now() });
  return data;
}
```

For server-rendered frameworks, rely on the built-in HTTP caching:

- **Persisted queries via GET** are CDN-cacheable (default `s-maxage: 7200`)
- **Next.js ISR** (`revalidate: 60`) re-fetches stale pages in the background
- **SvelteKit/Nuxt** server routes can set `Cache-Control` headers from AEM responses

### Preview Mode and Draft Content

Preview mode connects to the AEM Author tier (or Preview tier) to show unpublished content.

```javascript
// Next.js preview API route: pages/api/preview.js
export default function handler(req, res) {
  const { secret, slug } = req.query;
  if (secret !== process.env.PREVIEW_SECRET) {
    return res.status(401).json({ message: 'Invalid token' });
  }
  res.setPreviewData({});
  res.redirect(`/adventures/${slug}`);
}

// In getStaticProps, detect preview
export async function getStaticProps({ params, preview = false }) {
  const client = new AEMHeadless({
    serviceURL: preview
      ? process.env.AEM_AUTHOR_HOST   // Author tier for draft content
      : process.env.AEM_HOST,          // Publish tier for live content
    auth: preview ? process.env.AEM_PREVIEW_TOKEN : undefined,
  });

  const { data } = await client.runPersistedQuery(
    'wknd-shared/adventure-by-slug',
    { slug: params.slug }
  );

  return { props: { adventure: data?.adventureBySlug?.item, preview } };
}
```

### Multi-Environment Setup

AEM as a Cloud Service provides three tiers: Author, Publish, and Preview.

| Tier | URL Pattern | Use Case |
|------|-------------|----------|
| Author | `https://author-p{id}-e{id}.adobeaemcloud.com` | Content authoring, preview, admin API |
| Publish | `https://publish-p{id}-e{id}.adobeaemcloud.com` | Live content delivery |
| Preview | `https://preview-p{id}-e{id}.adobeaemcloud.com` | Staged content review before publish |

```bash
# .env — multi-environment
AEM_AUTHOR_HOST=https://author-p12345-e67890.adobeaemcloud.com
AEM_PUBLISH_HOST=https://publish-p12345-e67890.adobeaemcloud.com
AEM_PREVIEW_HOST=https://preview-p12345-e67890.adobeaemcloud.com
AEM_SERVICE_CREDENTIALS=./auth/service-credentials.json
```

### Best Practices

1. **Always use persisted queries** — they execute via GET and are CDN-cacheable
2. **Use environment variables** for all AEM hosts, tokens, and credentials
3. **Never expose Author tier or service credentials to client-side code**
4. **Prepend AEM host to `_dynamicUrl`** — it returns only a relative path
5. **Use `preferWebp: true`** in `_assetTransform` for optimal image delivery
6. **Configure CORS with explicit origins** — never use wildcard `*` in production
7. **Use ISR or server-side rendering** for SEO-critical headless pages
8. **Implement exponential backoff** for retries against AEM APIs
9. **Separate Author/Publish/Preview clients** — do not mix environment configurations
10. **Use service credentials in production**, dev tokens only for local development

### Pitfalls

- Using dev tokens in production (they expire in 24 hours)
- Fetching from Author tier in client-side code (exposes internal URLs, requires auth)
- Missing CORS config causes silent failures on browser-based GraphQL calls
- Using `_path` instead of `_dynamicUrl` for images bypasses web-optimized delivery
- Not setting `revalidate` in Next.js SSG causes stale content indefinitely
- Hardcoding AEM host URLs instead of using environment variables
