---
title: Content Fragments — Models, Authoring & Delivery
impact: HIGH
impactDescription: Content Fragments are the foundation of AEM headless and structured content; correct model design and delivery patterns directly impact scalability, performance, and editorial efficiency
tags: content-fragments, headless, models, graphql, openapi, variations, sling-models, htl, webhooks, core-components
---

## Content Fragments — Models, Authoring & Delivery

Content Fragments are structured, channel-agnostic content managed as AEM Assets. They are authored against Content Fragment Models and delivered via GraphQL, OpenAPI REST, or rendered on pages through Sling Models and HTL.

---

### Content Fragment Models

Models define the schema for Content Fragments. They live under `/conf/<project>/settings/dam/cfm/models` and must be **enabled** in the Configuration Browser before use.

#### Available Field Types (Data Types)

| Data Type | GraphQL Type | Notes |
|-----------|-------------|-------|
| **Single-line text** | `String` | Supports regex validation, unique constraint |
| **Multi-line text** | `String` (with `html`, `plaintext`, `markdown`, `json` renditions) | Rich Text, Plain Text, or Markdown; default format configurable |
| **Number** | `Float` / `Int` | Validation for min/max values |
| **Boolean** | `Boolean` | Checkbox in editor |
| **Date and Time** | `Calendar` | Date only, time only, or combined |
| **Enumeration** | `String` | Rendered as checkbox, radio button, or dropdown |
| **Tags** | `[String]` | Fragment author selects from AEM tag taxonomy |
| **Content Reference** | `String` (path) | References any content/asset; image dimension and file size validation |
| **Fragment Reference** | Nested type | Links to other CF instances; can restrict to specific model(s) |
| **JSON Object** | `JSON` | Freeform JSON with syntax highlighting in editor |
| **Tab Placeholder** | *(none — ignored by GraphQL schema)* | Visual grouping only; organises editor tabs |

#### Key Field Properties

```
Property Name    — must match [A-Za-z0-9_] (auto-derived from Field Label,
                   or set manually; cannot be changed after creation)
Render As        — single value or multiple (min / max items configurable)
Required         — enforced in Content Fragment Editor
Translatable     — marks field for translation workflows; enables _locale
                   GraphQL filter
Unique           — prevents duplicate values across fragments of the same model
```

#### Validation Rules

- **Single-line text**: regex pattern (e.g. `^[A-Z]{2}-\\d{4}$` for a product code)
- **Number**: specific value range
- **Content Reference**: maximum file size (bytes), image width/height range (px)
- **Fragment Reference**: restrict to one or more allowed CF Models
- **Multiple fields**: minimum / maximum number of items

#### Model Design Best Practices

- **Flat over deep**: prefer flat models with Fragment References for composition; keep nesting to a maximum of 2–3 levels.
- **Reusable leaf models**: extract shared structures (address, SEO metadata, CTA) into dedicated models referenced from many parents.
- **Use Tab Placeholders**: group related fields visually so editors can navigate large models.
- **Enumeration for controlled vocabularies**: use enumerations or tags (not free-text) for values consumed by front-end filters.
- **Plan field names upfront**: `Property Name` cannot be renamed after creation without breaking GraphQL queries.

> **Model inheritance is NOT supported.** There is no extends/inherits mechanism between CF Models. Use Fragment References for composition.

---

### Content Fragment Variations

Every Content Fragment has a **Master** variation. Additional named variations hold tailored versions of the same content.

#### Creating a Variation

1. Open fragment in Content Fragment Editor.
2. Open the Variations side-panel.
3. Select **Create Variation** and set Title / Description.
4. The new variation is copied from **Master** (always Master, never from another variation).

#### Synchronizing with Master

When Master changes, a variation's multi-line text elements can be synchronised:

- Actions menu on the variation: **Sync current element with master**
- Visual diff shows added (green), removed (red), replaced (blue) content.
- **Limitation**: sync only works on **Multi-line text** fields; other field types must be updated manually.

#### Variation Capabilities

- Each variation can have **independent tags** for CDN caching, bulk operations, and search filtering.
- Variations can use different editing modes (Rich Text, Plain Text, Markdown) per element.
- Master **cannot be deleted**; named variations can.

---

### Content Fragment Metadata and Tags

Content Fragments support standard DAM metadata and AEM tags:

```
/content/dam/<project>/fragments/my-article
  jcr:content/
    metadata/
      cq:tags            — assigned tags (String[])
      dc:title           — DAM title
      dc:description     — DAM description
      dam:lastModifiedBy — last editor
```

Tags assigned to Master auto-propagate to new variations. Metadata can be extended via a custom **Metadata Schema** at `/libs/dam/content/schemaeditors/forms/contentfragment`.

---

### Versioning and Comparison

AEM auto-creates a version when editing begins (cookie-based token tracks the editing session). Authors can also:

- View version history in the editor timeline.
- Compare any two versions side-by-side.
- Revert to a previous version.

Auto-save interval is configurable:

```xml
<!-- /conf/<project>/settings/dam/cfm/jcr:content -->
<jcr:content
    jcr:primaryType="nt:unstructured"
    autoSaveInterval="{Long}600"/>  <!-- seconds; default 600 = 10 min -->
```

---

### Structured Content: Nested Fragments

#### Fragment Reference vs Content Reference

| Aspect | Fragment Reference | Content Reference |
|--------|-------------------|-------------------|
| **Target** | Another Content Fragment | Any content path or DAM asset |
| **GraphQL** | Returns nested object with all fields | Returns the path as a `String` |
| **Use case** | Structured composition (author → article) | Link to image, PDF, page |
| **Model restriction** | Can restrict to specific model(s) | Can restrict by MIME type / dimensions |

#### Nested Fragment Example (GraphQL)

```graphql
{
  articleByPath(_path: "/content/dam/mysite/articles/intro") {
    item {
      title
      body { html }
      author {                       # Fragment Reference
        name
        bio { plaintext }
        headshot { ... on ImageRef { _publishUrl } }
      }
      relatedArticles {              # Fragment Reference (multi)
        title
        _path
      }
    }
  }
}
```

---

### Rendering Content Fragments in HTL (Sling Model)

The recommended pattern for rendering Content Fragments on AEM pages uses the **Content Fragment Core Component** or a custom Sling Model backed by the `ContentFragment` API.

#### Using the Content Fragment Core Component

The Core Component resource type is `core/wcm/components/contentfragment/v1/contentfragment`.

Author dialog allows:
- Selecting a fragment path
- Choosing **Single element** (one multi-line text field) or **Multiple elements**
- Paragraph range control (for long text)

#### Custom Sling Model with CF API

```java
import com.adobe.cq.dam.cfm.ContentFragment;
import com.adobe.cq.dam.cfm.ContentElement;
import com.adobe.cq.dam.cfm.FragmentData;

@Model(adaptables = SlingHttpServletRequest.class,
       adapters = ArticleModel.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class ArticleModel {

    @Self
    private SlingHttpServletRequest request;

    @ValueMapValue(name = "fragmentPath")
    private String fragmentPath;

    private ContentFragment fragment;

    @PostConstruct
    protected void init() {
        if (fragmentPath != null) {
            Resource cfResource = request.getResourceResolver()
                .getResource(fragmentPath);
            if (cfResource != null) {
                fragment = cfResource.adaptTo(ContentFragment.class);
            }
        }
    }

    public String getTitle() {
        return fragment != null ? fragment.getTitle() : null;
    }

    public String getBody() {
        if (fragment == null) return null;
        ContentElement element = fragment.getElement("body");
        return element != null
            ? element.getContent()   // returns HTML for rich-text
            : null;
    }

    /** Access a specific variation */
    public String getSummaryForVariation(String variationName) {
        if (fragment == null) return null;
        ContentElement element = fragment.getElement("summary");
        if (element == null) return null;
        FragmentData variation = element.getVariation(variationName)
            .getValue();
        return variation != null ? variation.getValue(String.class) : null;
    }
}
```

#### HTL Template

```html
<sly data-sly-use.article="com.mysite.models.ArticleModel">
  <article>
    <h1>${article.title}</h1>
    <div class="article-body">${article.body @ context='html'}</div>
  </article>
</sly>
```

> For **GraphQL-based delivery** (headless SPA / external consumers), see [graphql-headless.md](graphql-headless.md).

---

### Content Fragment Delivery APIs

AEM Cloud Service offers three delivery mechanisms:

| API | Method | Caching | Best For |
|-----|--------|---------|----------|
| **GraphQL (Persisted Queries)** | `GET` | CDN-cacheable | SPAs, mobile apps, complex queries |
| **OpenAPI REST Delivery** | `GET` | CDN-cacheable (browser 5 min, CDN 1 h) | Simple JSON delivery, edge-cached |
| **CF Management OpenAPI** | `GET/POST/PUT/DELETE` | Not cached (author only) | CRUD from external systems |

#### OpenAPI for Content Fragment Delivery

The REST delivery API runs on AEM Edge Delivery Services and returns CF data as JSON:

```
GET https://publish-p<program>-e<env>.adobeaemcloud.com/adobe/sites/cf/fragments/<fragment-id>
```

Key characteristics:
- Rate limit: **200 requests/second** per environment (429 with `Retry-After` header on excess).
- Cache TTLs (non-configurable): browser 5 min, CDN 1 h, stale-while-revalidate 1 h, stale-on-error 24 h.
- Authentication via AEM CDN Edge keys.
- CORS configuration required; Dispatcher/GraphQL CORS settings do **not** apply.

#### OpenAPI for Content Fragment Management (CRUD)

The management API targets **author** instances only:

```
POST   /adobe/sites/cf/fragments         — create
GET    /adobe/sites/cf/fragments/{id}     — read
PUT    /adobe/sites/cf/fragments/{id}     — update
DELETE /adobe/sites/cf/fragments/{id}     — delete
```

- Requires authentication (Adobe IMS / service credentials).
- Disabled on publish by default.
- Replaces the legacy Assets HTTP API for CF management.

---

### AEM Eventing for Content Fragments (Webhooks)

AEM Cloud Service emits **CloudEvents** (JSON) when Content Fragments change. Subscribe via the Adobe Developer Console.

#### Event Types

| Event | Trigger |
|-------|---------|
| `aem.sites.contentFragment.created` | New CF created |
| `aem.sites.contentFragment.modified` | CF updated |
| `aem.sites.contentFragment.deleted` | CF deleted |
| `aem.sites.contentFragment.published` | CF published |
| `aem.sites.contentFragment.unpublished` | CF unpublished |

#### Delivery Methods

- **Push**: Webhook URL, Adobe I/O Runtime Action, or Amazon EventBridge.
- **Pull**: Adobe Developer Journaling API (polling).

#### Webhook Example (Node.js)

```javascript
// Adobe I/O Runtime Action — receives CF events
async function main(params) {
  const event = params;

  if (event.type === 'aem.sites.contentFragment.published') {
    const fragmentPath = event.data?.path;
    const fragmentModel = event.data?.modelId;

    // Trigger downstream cache purge / reindex / notification
    await invalidateCDN(fragmentPath);

    return { statusCode: 200 };
  }
  return { statusCode: 204 };
}
```

---

### Core Components for Content Fragments

#### Content Fragment Component

- **Resource type**: `core/wcm/components/contentfragment/v1/contentfragment`
- Displays a **single** Content Fragment on a page.
- Configure: fragment path, element selection (single or multiple), paragraph range.
- Supports AEM Style System.

#### Content Fragment List Component

- **Resource type**: `core/wcm/components/contentfragmentlist/v2/contentfragmentlist`
- Displays **multiple** fragments filtered by model, parent path, and tags.

Configuration options:
| Property | Description |
|----------|-------------|
| **Model** | CF Model to filter by (required) |
| **Parent Path** | DAM folder to scope results |
| **Tags** | Further filter by assigned tags |
| **Order By** | Any text/number/date field from the model |
| **Sort Order** | Ascending or Descending |
| **Max Items** | Limit result count (blank = all) |
| **Elements** | Select which fields to include in output |

---

### Organizational Patterns: Assets vs Sites

Content Fragments live under `/content/dam` (Assets). Organize by:

```
/content/dam/<project>/
  fragments/
    articles/         — by content type
      en/             — by locale
    authors/
    categories/
  images/
```

**Key rules**:
- CF Models are under `/conf/<project>/settings/dam/cfm/models` — the project Configuration must be assigned to the DAM folder via **Cloud Configuration**.
- Fragments inherit the Configuration of their parent folder.
- Use folders (not tags alone) for access control — DAM folder permissions govern who can author which fragments.

---

### Best Practices

1. **Design models for query patterns** — anticipate GraphQL filters and structure fields accordingly.
2. **Limit nesting depth to 2–3 levels** — deep nesting multiplies GraphQL response size and slows delivery.
3. **Use Fragment References for composition, Content References for media** — do not abuse Fragment References to link unrelated content.
4. **Create variations only when content genuinely differs** — for layout-only differences, handle presentation in the consuming channel.
5. **Publish fragments atomically** — when a parent fragment references children, publish children first to avoid broken references on the delivery tier.
6. **Use the OpenAPI delivery endpoint for simple JSON needs** — avoid GraphQL overhead when you only need one fragment by ID.
7. **Subscribe to CF events for cache invalidation** — use webhooks to purge CDN or trigger rebuilds when content changes.
8. **Name fields deliberately** — `Property Name` is immutable; changing it later requires a new model + content migration.

### Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| **Over-nesting** (4+ levels) | GraphQL queries become slow, payload bloated | Flatten; use 2-level max in practice |
| **Circular references** (A → B → A) | Infinite recursion in GraphQL, broken delivery | Enforce DAG structure; reference "down" only |
| **Too many variations** | Editorial confusion, sync overhead | Use tags or separate fragments instead |
| **Free-text where enumeration fits** | Inconsistent data, broken front-end filters | Use Enumeration or Tags for controlled lists |
| **Single mega-model** | Hard to maintain, poor query performance | Decompose into focused models + references |
| **Storing rendered HTML in CF** | Defeats channel-agnostic purpose | Store structured data; render in consuming layer |
| **Ignoring field uniqueness** | Duplicate content, ambiguous queries | Enable Unique constraint on identifier fields |
