---
title: GraphQL & Headless Content Delivery
impact: HIGH
impactDescription: Correct GraphQL patterns enable CDN caching and prevent performance degradation — POST queries bypass CDN entirely, and unoptimized queries can bring down publish instances
tags: graphql, content-fragments, headless, persisted-queries, api, caching, filtering, pagination, sdk
---

## GraphQL & Headless Content Delivery

AEM provides a GraphQL API for Content Fragments. Persisted queries are the recommended approach for production — they execute via GET and are CDN-cacheable. POST queries bypass CDN entirely.

---

### 1. Persisted Queries

#### Creating Persisted Queries

```bash
# Via PUT to the GraphQL endpoint
curl -X PUT \
  "https://author-p12345-e67890.adobeaemcloud.com/graphql/execute.json/mysite/adventures-all" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "query": "{ adventureList { items { _path title activity tripLength } } }" }'
```

Or via the GraphQL Persisted Queries editor in AEM (`/aem/graphql/persisted-queries.html`).

#### Executing Persisted Queries

```
GET /graphql/execute.json/<config>/<query-name>[;param=value]
```

```bash
# Simple query
GET /graphql/execute.json/mysite/adventures-all

# With parameters
GET /graphql/execute.json/mysite/adventures-by-activity;activity=Surfing

# Multiple parameters
GET /graphql/execute.json/mysite/adventures-filtered;activity=Surfing;minLength=3
```

#### Persisted Query Cache TTLs

**Default:**
- Browser: 60 seconds (`max-age`)
- CDN/Dispatcher: 7200 seconds (`s-maxage`)

**Custom TTL per query:**

```bash
# Set cache headers when creating the persisted query
curl -X PUT \
  "https://author-host/graphql/execute.json/mysite/adventures-all" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ adventureList { items { title } } }",
    "cache-control": { "max-age": 300, "s-maxage": 3600 }
  }'
```

---

### 2. Query Patterns

#### Single Item by Path

```graphql
{
  adventureByPath(_path: "/content/dam/mysite/en/adventures/bali") {
    item {
      _path
      title
      description { html plaintext }
      primaryImage { ... on ImageRef { _dynamicUrl _path width height } }
    }
  }
}
```

#### List with Filtering

```graphql
{
  adventureList(
    filter: {
      activity: { _expressions: [{ value: "Surfing" }] }
      tripLength: { _expressions: [{ value: "2", _operator: GREATER }] }
    }
    sort: "title ASC"
    _locale: "en"
  ) {
    items {
      _path
      title
      activity
      tripLength
    }
  }
}
```

#### Parameterized Persisted Query

```graphql
# Query definition with variables
query AdventuresByActivity($activity: String!, $limit: Int = 10) {
  adventureList(
    filter: { activity: { _expressions: [{ value: $activity }] } }
    _limit: $limit
    sort: "title ASC"
  ) {
    items { _path title activity tripLength }
  }
}
```

```bash
# Execution with parameters
GET /graphql/execute.json/mysite/adventures-by-activity;activity=Surfing;limit=5
```

---

### 3. Filter Operators

| Operator | Types | Description | Example |
|----------|-------|-------------|---------|
| `EQUALS` | String, ID, Boolean | Exact match | `{ value: "Surfing" }` |
| `EQUALS_NOT` | String, ID, Boolean | Not equal | `{ value: "Draft", _operator: EQUALS_NOT }` |
| `CONTAINS` | String | Substring match | `{ value: "surf", _operator: CONTAINS }` |
| `CONTAINS_NOT` | String | Not contains | `{ value: "test", _operator: CONTAINS_NOT }` |
| `STARTS_WITH` | String, ID | Prefix match | `{ value: "Sur", _operator: STARTS_WITH }` |
| `GREATER` | Int, Float | Greater than | `{ value: "5", _operator: GREATER }` |
| `GREATER_EQUAL` | Int, Float | Greater or equal | `{ value: "5", _operator: GREATER_EQUAL }` |
| `LOWER` | Int, Float | Less than | `{ value: "10", _operator: LOWER }` |
| `LOWER_EQUAL` | Int, Float | Less or equal | `{ value: "10", _operator: LOWER_EQUAL }` |
| `AT` | Calendar, Date | Exact date | `{ value: "2024-01-01", _operator: AT }` |
| `BEFORE` | Calendar, Date | Before date | `{ value: "2024-06-01", _operator: BEFORE }` |
| `AFTER` | Calendar, Date | After date | `{ value: "2024-01-01", _operator: AFTER }` |
| `AT_NOT` | Calendar, Date | Not at date | `{ value: "2024-01-01", _operator: AT_NOT }` |

#### Case Insensitive

```graphql
filter: {
  title: { _expressions: [{ value: "bali", _operator: CONTAINS, _ignoreCase: true }] }
}
```

#### OR Logic

```graphql
{
  authorList(filter: {
    lastName: {
      _logOp: OR
      _expressions: [
        { value: "sjö", _operator: CONTAINS, _ignoreCase: true },
        { value: "Provo" }
      ]
    }
  }) {
    items { lastName firstName }
  }
}
```

#### Null Checks

```graphql
# Find items where field IS set
filter: { featuredImage: { _expressions: [{ _operator: EXISTS }] } }

# Find items where field is NOT set
filter: { featuredImage: { _expressions: [{ _operator: EXISTS_NOT }] } }
```

---

### 4. Pagination

#### Offset-Based Pagination

```graphql
{
  adventureList(
    _offset: 0
    _limit: 10
    sort: "title ASC"
  ) {
    items { title }
  }
}
```

```javascript
// Client-side pagination
async function loadPage(offset = 0, limit = 10) {
  const response = await fetch(
    `/graphql/execute.json/mysite/adventures-paginated;offset=${offset};limit=${limit}`
  );
  return response.json();
}
```

#### Cursor-Based Pagination (Recommended for Large Datasets)

```graphql
{
  adventurePaginated(first: 10, after: "YXJyYXljb25uZWN0aW9uOjk=") {
    edges {
      cursor
      node {
        _path
        title
        activity
      }
    }
    pageInfo {
      endCursor
      hasNextPage
      hasPreviousPage
      startCursor
    }
    totalCount
  }
}
```

```javascript
// Infinite scroll with cursor pagination
let endCursor = null;

async function loadMore() {
  const params = endCursor ? `;after=${endCursor}` : '';
  const response = await fetch(
    `/graphql/execute.json/mysite/adventures-cursor;first=10${params}`
  );
  const { data } = await response.json();
  const { edges, pageInfo } = data.adventurePaginated;

  renderItems(edges.map(e => e.node));
  endCursor = pageInfo.endCursor;

  if (!pageInfo.hasNextPage) {
    hideLoadMoreButton();
  }
}
```

---

### 5. Nested Fragment References

```graphql
{
  articleByPath(_path: "/content/dam/mysite/articles/intro") {
    item {
      title
      body { html }
      # Nested Content Fragment reference
      author {
        ... on AuthorModel {
          firstName
          lastName
          bio { html }
          profilePicture { ... on ImageRef { _dynamicUrl } }
        }
      }
      # Multi-value fragment reference
      relatedArticles {
        ... on ArticleModel {
          _path
          title
          summary { plaintext }
        }
      }
    }
  }
}
```

---

### 6. Web-Optimized Image Delivery

```graphql
{
  articleList(
    _assetTransform: {
      format: WEBP
      preferWebp: true
      size: { width: 800 }
      quality: 80
    }
  ) {
    items {
      title
      featuredImage {
        ... on ImageRef {
          _dynamicUrl     # Web-optimized URL
          _path           # Original DAM path
          width
          height
          mimeType
        }
      }
    }
  }
}
```

```html
<!-- Use _dynamicUrl in frontend -->
<img src="${article.featuredImage._dynamicUrl}"
     width="${article.featuredImage.width}"
     height="${article.featuredImage.height}"
     alt="${article.title}"
     loading="lazy"/>
```

---

### 7. GraphQL Endpoint Configuration

```
Endpoints are configured per Content Fragment configuration:
  /conf/<config>/settings/graphql/endpoints

Default endpoint:
  /content/cq:graphql/<config>/endpoint
```

**CORS configuration for headless frontend:**

```json
// com.adobe.granite.cors.impl.CORSPolicyImpl~graphql.cfg.json
{
    "alloworigin": ["https://www.mysite.com", "https://app.mysite.com"],
    "allowedpaths": ["/graphql/execute.json/.*", "/content/cq:graphql/.*"],
    "allowedmethods": ["GET", "OPTIONS"],
    "maxage:Integer": 86400,
    "allowedheaders": ["Content-Type", "Authorization"],
    "supportedheaders": [""]
}
```

---

### 8. AEM Headless SDK

#### JavaScript/Node.js SDK

```javascript
import AEMHeadless from '@adobe/aem-headless-client-js';

const client = new AEMHeadless({
  serviceURL: 'https://publish-p123-e456.adobeaemcloud.com',
  endpoint: '/content/cq:graphql/mysite/endpoint',
  auth: process.env.AEM_TOKEN, // Optional: for protected content
});

// Execute persisted query
const { data } = await client.runPersistedQuery(
  'mysite/adventures-by-activity',
  { activity: 'Surfing' }
);

// Execute ad-hoc query (dev only — not CDN-cacheable)
const { data: devData } = await client.runQuery(`{
  adventureList { items { title } }
}`);
```

#### React Hook

```javascript
import { useEffect, useState } from 'react';
import AEMHeadless from '@adobe/aem-headless-client-js';

const client = new AEMHeadless({ serviceURL: process.env.NEXT_PUBLIC_AEM_HOST });

export function useAdventures(activity) {
  const [adventures, setAdventures] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    client.runPersistedQuery('mysite/adventures-by-activity', { activity })
      .then(({ data }) => setAdventures(data.adventureList.items))
      .catch(setError)
      .finally(() => setLoading(false));
  }, [activity]);

  return { adventures, loading, error };
}
```

#### Next.js ISR with GraphQL

```javascript
// pages/adventures/[slug].js
export async function getStaticProps({ params }) {
  const { data } = await client.runPersistedQuery(
    'mysite/adventure-by-slug',
    { slug: params.slug }
  );

  return {
    props: { adventure: data.adventureByPath.item },
    revalidate: 60, // ISR: revalidate every 60 seconds
  };
}
```

---

### 9. Helper Fields

| Field | Description |
|-------|-------------|
| `_path` | Repository path |
| `_id` | UUID (stable across moves) |
| `_metadata` | System metadata (stringMetadata, intMetadata, etc.) |
| `_variations` | Available CF variations |
| `_tags` | Content tags |
| `_references` | All referenced content |
| `_model` | Content Fragment model info |

---

### 10. Performance Optimization

#### Query Complexity

```graphql
# WRONG — deeply nested query with multiple references
{
  articleList {
    items {
      author { relatedAuthors { articles { author { bio { html } } } } }
    }
  }
}
# Exponential joins — can time out

# CORRECT — flat structure, fetch related data in separate queries
{
  articleList { items { _path title authorRef } }
}
# Then fetch author details in a second query if needed
```

#### Limits

- Maximum 1M characters per query
- Maximum 15K tokens
- Default `_limit`: 20 items
- Maximum `_limit`: configurable (default 100)

---

### 11. Anti-Patterns

#### POST Queries in Production

```javascript
// WRONG — POST bypasses CDN entirely
const response = await fetch('/graphql/global/endpoint.json', {
  method: 'POST',
  body: JSON.stringify({ query: '{ adventureList { items { title } } }' })
});

// CORRECT — use persisted queries (GET, CDN-cacheable)
const response = await fetch('/graphql/execute.json/mysite/adventures-all');
```

#### Not URL-Encoding Query Variables

```javascript
// WRONG — special characters break URL parsing
fetch(`/graphql/execute.json/mysite/search;query=Rock & Roll`);

// CORRECT — encode special characters
fetch(`/graphql/execute.json/mysite/search;query=${encodeURIComponent('Rock & Roll')}`);
```

#### Fetching All Fields

```graphql
# WRONG — over-fetching wastes bandwidth and processing
{ adventureList { items { _path title description { html plaintext markdown } activity
    tripLength price difficulty itinerary { html } primaryImage { ... on ImageRef {
    _path _dynamicUrl width height mimeType } } } } }

# CORRECT — request only what you display
{ adventureList { items { _path title activity primaryImage { ... on ImageRef { _dynamicUrl } } } } }
```

#### Client-Side Filtering Instead of Server-Side

```javascript
// WRONG — fetch all, filter in JS
const all = await fetchAllAdventures();
const surfing = all.filter(a => a.activity === 'Surfing');

// CORRECT — filter in GraphQL query
const surfing = await client.runPersistedQuery(
  'mysite/adventures-by-activity', { activity: 'Surfing' }
);
```
