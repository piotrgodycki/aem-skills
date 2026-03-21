---
title: Content Fragment OpenAPI & AEM Eventing
impact: HIGH
impactDescription: OpenAPI and eventing patterns determine API reliability, security, and real-time integration capabilities for headless architectures
tags: openapi, content-fragments, api, events, webhooks, adobe-io, app-builder, rest, crud
---

## Content Fragment OpenAPI & AEM Eventing

AEM as a Cloud Service provides OpenAPI-based REST APIs for Content Fragment management and delivery, plus a cloud-native eventing system (Adobe I/O Events) for real-time integrations. The legacy Assets HTTP API for Content Fragments is deprecated in favor of these OpenAPIs.

### Content Fragment Management OpenAPI (Admin API)

The CF Management API enables CRUD operations on Content Fragments and Content Fragment Models on the **Author tier only**. It is disabled on Publish by default.

Full specification: `https://developer.adobe.com/experience-cloud/experience-manager-apis/api/stable/sites/`

#### Authentication

All Management API calls require OAuth authentication provisioned through Adobe Developer Console:

- **OAuth Server-to-Server** — for backend service integrations
- **OAuth Web App** — for traditional web applications with user context
- **OAuth Single Page App** — for browser-based applications

Setup requires:
1. Product Profile assignment granting API access
2. Adobe Developer Console project with OAuth credentials
3. Client ID registration via YAML config deployed through Cloud Manager

#### CRUD Operations

**Create a Content Fragment:**

```bash
# Correct: use Management OpenAPI on Author tier
curl -X POST \
  "https://author-p123-e456.adobeaemcloud.com/adobe/sites/cf/fragments" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Bali Surf Camp",
    "description": "A surfing adventure in Bali",
    "modelId": "L2NvbmYvd2tuZC1zaGFyZWQvc2V0dGluZ3MvZGFtL2NmbS9tb2RlbHMvYWR2ZW50dXJl",
    "parentPath": "/content/dam/wknd-shared/en/adventures",
    "fields": [
      { "name": "title", "type": "text", "value": "Bali Surf Camp" },
      { "name": "activity", "type": "text", "value": "Surfing" },
      { "name": "price", "type": "number", "value": 1500 }
    ]
  }'
```

**Read a Content Fragment:**

```bash
curl -X GET \
  "https://author-p123-e456.adobeaemcloud.com/adobe/sites/cf/fragments/{fragment-id}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Accept: application/json"
```

**Update a Content Fragment:**

```bash
curl -X PUT \
  "https://author-p123-e456.adobeaemcloud.com/adobe/sites/cf/fragments/{fragment-id}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "If-Match: \"etag-value\"" \
  -d '{
    "fields": [
      { "name": "price", "type": "number", "value": 1800 }
    ]
  }'
```

**Delete a Content Fragment:**

```bash
curl -X DELETE \
  "https://author-p123-e456.adobeaemcloud.com/adobe/sites/cf/fragments/{fragment-id}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

**Publish a Content Fragment:**

```bash
curl -X POST \
  "https://author-p123-e456.adobeaemcloud.com/adobe/sites/cf/fragments/{fragment-id}/publish" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

#### Search and Filter

```bash
# List fragments by model
curl -X GET \
  "https://author-p123-e456.adobeaemcloud.com/adobe/sites/cf/fragments?modelId={model-id}&offset=0&limit=20" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"

# Search by path prefix
curl -X GET \
  "https://author-p123-e456.adobeaemcloud.com/adobe/sites/cf/fragments?parentPath=/content/dam/wknd-shared/en/adventures" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

### Content Fragment Delivery OpenAPI

The Delivery API is optimized for **live content delivery** in JSON format on the **Publish and Preview tiers**. It integrates with Fastly CDN with automatic cache invalidation.

#### Caching Defaults

| Layer | TTL |
|-------|-----|
| Browser cache | 5 minutes |
| CDN (Fastly) | 1 hour |
| Stale-while-revalidate | up to 1 hour |
| Stale-if-error | up to 1 day |

Cache is automatically invalidated when content is published.

#### Rate Limits

- **200 requests per second** per environment
- Returns HTTP `429 Too Many Requests` with `Retry-After` header when exceeded

#### Authentication

Delivery API supports **AEM CDN Edge key** authentication. Repository-level ACL-based authorization is not currently supported.

#### CORS for Delivery API

Delivery API CORS must be configured separately from GraphQL CORS. Dispatcher-level GraphQL CORS settings do not apply.

### Delivery API vs Management API

| Aspect | Delivery API | Management API |
|--------|-------------|----------------|
| **Tier** | Publish, Preview | Author only |
| **Operations** | Read-only (GET) | Full CRUD |
| **Authentication** | CDN Edge key | OAuth (Server-to-Server, Web App, SPA) |
| **Caching** | Fastly CDN integrated | No CDN caching |
| **Rate limit** | 200 req/s per environment | Subject to Author tier limits |
| **Use case** | Frontend content delivery | Content management integrations |

### Assets HTTP API (Legacy)

The Assets HTTP API (`/api/assets/`) provides DAM CRUD operations. For Content Fragments, **migrate to the Management OpenAPI**.

```bash
# Legacy — deprecated for Content Fragments
curl -X GET \
  "https://author-p123-e456.adobeaemcloud.com/api/assets/wknd-shared/en/adventures/bali-surf-camp.json" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"

# New — use Management OpenAPI instead
curl -X GET \
  "https://author-p123-e456.adobeaemcloud.com/adobe/sites/cf/fragments/{fragment-id}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

The Assets HTTP API remains valid for general DAM asset operations (binaries, folders, metadata).

### AEM Eventing / Adobe I/O Events

AEM Eventing is a cloud-native publish-subscribe system. AEM as a Cloud Service produces events and sends them to Adobe I/O Events, which distributes them to subscribers. Events follow the CloudEvents specification.

#### Architecture

```
AEM as a Cloud Service  -->  Adobe I/O Events (broker)  -->  Event Consumers
  (event producer)              |                              - Webhooks
                                |                              - Adobe I/O Runtime
                                |                              - Amazon EventBridge
                                |                              - Journaling API (pull)
```

#### Event Types for Content Fragments

| Event Type | Trigger |
|------------|---------|
| `aem.sites.contentFragment.created` | Content Fragment created |
| `aem.sites.contentFragment.modified` | Content Fragment updated |
| `aem.sites.contentFragment.deleted` | Content Fragment deleted |
| `aem.sites.contentFragment.published` | Content Fragment published |
| `aem.sites.contentFragment.unpublished` | Content Fragment unpublished |

#### Event Payload (CloudEvents Format)

```json
{
  "specversion": "1.0",
  "id": "83b0eac0-56d6-4499-afa6-4dc58ff6ac7f",
  "source": "acct:aem-p63947-e1249010@adobe.com",
  "type": "aem.sites.contentFragment.modified",
  "datacontenttype": "application/json",
  "dataschema": "https://ns.adobe.com/xdm/aem/sites/events/content-fragment-modified.json",
  "time": "2025-07-24T13:53:23.994Z",
  "data": {
    "user": {
      "imsUserId": "ims-933E1F8A631CAA0F0A495E53@tech.e",
      "principalId": "user@adobe.com",
      "displayName": "John Doe"
    },
    "path": "/content/dam/wknd-shared/en/adventures/bali-surf-camp",
    "sourceUrl": "https://author-p63947-e1249010.adobeaemcloud.com",
    "model": {
      "id": "L2NvbmYvd2tuZC1zaGFyZWQvc2V0dGluZ3MvZGFtL2NmbS9tb2RlbHMvYWR2ZW50dXJl",
      "path": "/conf/wknd-shared/settings/dam/cfm/models/adventure"
    },
    "id": "9e1e9835-64c8-42dc-9d36-fbd59e28f753",
    "tags": ["wknd-shared:region/nam/united-states"],
    "properties": [
      { "name": "price", "changeType": "modified" }
    ]
  }
}
```

#### Request Headers for Validation

```
content-type: application/cloudevents+json; charset=UTF-8
x-adobe-event-code: aem.sites.contentFragment.modified
x-adobe-event-id: b555a1b1-935b-4541-b410-1915775338b5
x-adobe-digital-signature-1: <signature>
x-adobe-digital-signature-2: <signature>
x-adobe-public-key1-path: /prod/keys/pub-key-<id>.pem
x-adobe-public-key2-path: /prod/keys/pub-key-<id>.pem
```

#### Webhook Registration and Handler

Register webhooks through Adobe Developer Console. The webhook must handle the challenge probe.

```javascript
// Express webhook handler
const express = require('express');
const app = express();

app.use(express.json({ type: 'application/cloudevents+json' }));

app.post('/webhook/aem-events', (req, res) => {
  // Handle Adobe I/O challenge probe (registration verification)
  if (req.body.challenge) {
    console.log('Challenge probe received');
    return res.json({ challenge: req.body.challenge });
  }

  const event = req.body;
  console.log(`Event received: ${event.type} for ${event.data?.path}`);

  switch (event.type) {
    case 'aem.sites.contentFragment.modified':
      handleContentUpdate(event.data);
      break;
    case 'aem.sites.contentFragment.published':
      handleContentPublished(event.data);
      break;
    case 'aem.sites.contentFragment.deleted':
      handleContentDeleted(event.data);
      break;
  }

  res.status(200).json({ message: 'Event processed' });
});

async function handleContentUpdate(data) {
  // Invalidate CDN/cache for the updated content
  await invalidateCache(data.path);
}

async function handleContentPublished(data) {
  // Trigger ISR revalidation on the frontend
  await fetch(`${process.env.FRONTEND_URL}/api/revalidate`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      secret: process.env.REVALIDATION_SECRET,
      path: data.path,
    }),
  });
}
```

#### Adobe I/O Runtime Action for Event Processing

```javascript
// actions/aem-event-handler/index.js
const { Core } = require('@adobe/aio-sdk');
const auth = require('@adobe/jwt-auth');
const fetch = require('node-fetch');

async function main(params) {
  const logger = Core.Logger('aem-event-handler', { level: 'info' });

  // Handle challenge probe
  if (params.challenge) {
    logger.info('Challenge probe detected');
    return {
      statusCode: 200,
      body: JSON.stringify({ challenge: params.challenge }),
    };
  }

  // Process the AEM event
  const event = params;
  logger.info(`Processing event: ${event.type}`);

  // Validate event type
  if (event.type !== 'aem.sites.contentFragment.modified') {
    return { statusCode: 200, body: 'Event type not handled' };
  }

  // Optionally fetch full fragment details from AEM Author
  const accessToken = await getAccessToken(params);
  const fragmentDetails = await fetch(
    `${event.data.sourceUrl}/adobe/sites/cf/fragments/${event.data.id}`,
    {
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Content-Type': 'application/json',
      },
    }
  ).then((res) => res.json());

  // Store or forward the event data
  const filesLib = require('@adobe/aio-lib-files');
  const files = await filesLib.init();
  await files.write(
    `events/${new Date().toISOString().split('T')[0]}/${event.id}.json`,
    JSON.stringify({ event, fragmentDetails })
  );

  return {
    statusCode: 200,
    body: JSON.stringify({ message: 'Event processed', eventId: event.id }),
  };
}

async function getAccessToken(params) {
  const { access_token } = await auth({
    clientId: params.AEM_CLIENT_ID,
    technicalAccountId: params.AEM_TECHNICAL_ACCOUNT_ID,
    orgId: params.AEM_ORG_ID,
    clientSecret: params.AEM_CLIENT_SECRET,
    privateKey: params.AEM_PRIVATE_KEY,
    metaScopes: [params.AEM_META_SCOPE],
    ims: 'https://ims-na1.adobelogin.com',
  });
  return access_token;
}

exports.main = main;
```

#### Event-Driven Frontend: Cache Invalidation

```javascript
// Next.js: pages/api/revalidate.js — on-demand ISR via AEM events
export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method not allowed' });
  }

  const { secret, path } = req.body;
  if (secret !== process.env.REVALIDATION_SECRET) {
    return res.status(401).json({ message: 'Invalid secret' });
  }

  try {
    // Convert AEM content path to frontend route
    // /content/dam/wknd-shared/en/adventures/bali-surf-camp -> /adventures/bali-surf-camp
    const slug = path.split('/').pop();
    await res.revalidate(`/adventures/${slug}`);
    return res.json({ revalidated: true, slug });
  } catch (err) {
    return res.status(500).json({ message: 'Revalidation failed' });
  }
}
```

#### Journaling API (Pull-Based)

For consumers that cannot receive webhooks, the Journaling API provides a pull-based approach.

```javascript
async function pollJournal(journalUrl, accessToken, lastPosition) {
  const url = lastPosition
    ? `${journalUrl}?since=${lastPosition}`
    : journalUrl;

  const response = await fetch(url, {
    headers: { Authorization: `Bearer ${accessToken}` },
  });
  const data = await response.json();

  // Process events
  for (const event of data.events || []) {
    await processEvent(event);
  }

  // Return position for next poll
  return data._links?.next?.href || lastPosition;
}
```

### Adobe App Builder Integration

App Builder provides a complete framework for building AEM event-driven applications with I/O Runtime, file storage, and state management.

```yaml
# app.config.yaml — App Builder project with AEM event registration
application:
  actions:
    aem-event-handler:
      function: actions/aem-event-handler/index.js
      web: 'yes'
      runtime:
        kind: 'nodejs:18'
      inputs:
        AEM_CLIENT_ID: $AEM_CLIENT_ID
        AEM_TECHNICAL_ACCOUNT_ID: $AEM_TECHNICAL_ACCOUNT_ID
        AEM_ORG_ID: $AEM_ORG_ID
        AEM_CLIENT_SECRET: $AEM_CLIENT_SECRET
        AEM_PRIVATE_KEY: $AEM_PRIVATE_KEY
      annotations:
        require-adobe-auth: true
  events:
    registrations:
      aem-cf-events:
        description: AEM Content Fragment events
        events_of_interest:
          - provider_metadata: aem
            event_codes:
              - aem.sites.contentFragment.created
              - aem.sites.contentFragment.modified
              - aem.sites.contentFragment.deleted
              - aem.sites.contentFragment.published
        runtime_action: aem-event-handler
```

### Pagination in APIs

**Correct — paginated requests:**

```javascript
async function fetchAllFragments(modelId) {
  const fragments = [];
  let offset = 0;
  const limit = 20;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(
      `${AEM_HOST}/adobe/sites/cf/fragments?modelId=${modelId}&offset=${offset}&limit=${limit}`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    const data = await response.json();
    fragments.push(...(data.items || []));

    offset += limit;
    hasMore = data.items?.length === limit;
  }

  return fragments;
}
```

**Incorrect — fetching without pagination:**

```javascript
// Fetches unbounded results — may timeout or hit memory limits
const response = await fetch(
  `${AEM_HOST}/adobe/sites/cf/fragments?modelId=${modelId}`,
  { headers: { Authorization: `Bearer ${token}` } }
);
```

### Bulk Operations

For bulk updates, batch requests and handle rate limits with backoff.

```javascript
async function bulkUpdateFragments(fragments, updateFn) {
  const BATCH_SIZE = 10;
  const DELAY_MS = 200; // Respect rate limits

  for (let i = 0; i < fragments.length; i += BATCH_SIZE) {
    const batch = fragments.slice(i, i + BATCH_SIZE);
    const results = await Promise.allSettled(
      batch.map((fragment) => updateFn(fragment))
    );

    // Log failures
    results.forEach((result, idx) => {
      if (result.status === 'rejected') {
        console.error(`Failed to update ${batch[idx].id}:`, result.reason);
      }
    });

    // Delay between batches to avoid rate limiting
    if (i + BATCH_SIZE < fragments.length) {
      await new Promise((resolve) => setTimeout(resolve, DELAY_MS));
    }
  }
}
```

### API Versioning and Stability

- OpenAPI specifications are versioned and published at `developer.adobe.com`
- The **Management OpenAPI** replaces the deprecated Assets HTTP API for Content Fragments
- The **Delivery OpenAPI** replaces the deprecated Assets HTTP API for content delivery
- Monitor the `Deprecation` and `Sunset` HTTP headers in API responses for migration timelines
- Pin integrations to specific API versions where available

### Anti-Patterns

**Polling instead of events:**

```javascript
// Incorrect: polling AEM for changes wastes resources and adds latency
setInterval(async () => {
  const fragments = await fetchAllFragments();
  const changed = diff(fragments, previousFragments);
  if (changed.length > 0) processChanges(changed);
  previousFragments = fragments;
}, 5000);
```

```javascript
// Correct: subscribe to AEM events via webhooks
// Register webhook in Adobe Developer Console, then handle events:
app.post('/webhook/aem-events', (req, res) => {
  if (req.body.challenge) return res.json({ challenge: req.body.challenge });
  processEvent(req.body);
  res.sendStatus(200);
});
```

**Exposing Admin API to client-side code:**

```javascript
// Incorrect: Management API credentials in browser code
const response = await fetch(
  'https://author-p123-e456.adobeaemcloud.com/adobe/sites/cf/fragments',
  {
    headers: {
      Authorization: `Bearer ${clientSideToken}`, // Never expose Author credentials
    },
  }
);
```

```javascript
// Correct: use Delivery API on Publish tier for client-side, or proxy through backend
const response = await fetch(
  'https://publish-p123-e456.adobeaemcloud.com/adobe/sites/cf/fragments/{id}',
  {
    headers: { 'X-AEM-Edge-Key': process.env.NEXT_PUBLIC_CDN_EDGE_KEY },
  }
);
```

**No pagination on list endpoints:**

```javascript
// Incorrect: unbounded fetch — slow, may timeout
const all = await fetch(`${AEM_HOST}/adobe/sites/cf/fragments`);

// Correct: always paginate
const page = await fetch(`${AEM_HOST}/adobe/sites/cf/fragments?offset=0&limit=20`);
```

### Best Practices

1. **Use the Management OpenAPI** for CRUD — the Assets HTTP API for Content Fragments is deprecated
2. **Use the Delivery OpenAPI** for frontend content fetching on Publish/Preview tiers
3. **Use events (webhooks) instead of polling** for real-time change detection
4. **Always paginate** list API calls with `offset` and `limit` parameters
5. **Implement retry with backoff** for `429 Too Many Requests` responses
6. **Use `If-Match` (ETags) for updates** to prevent lost-update conflicts
7. **Keep Management API server-side only** — never expose Author credentials to browsers
8. **Validate webhook signatures** using the `x-adobe-digital-signature` headers
9. **Use App Builder** for complex event processing workflows with state management
10. **Handle the challenge probe** in all webhook and Runtime action handlers
