---
title: Adobe Client Data Layer (ACDL) — Complete Implementation Guide
impact: HIGH
impactDescription: Incorrect data layer implementation causes missing analytics data, broken tag management, lost conversion tracking, and unreliable reporting across the entire site
tags: acdl, data-layer, analytics, adobe-launch, tags, google-tag-manager, tracking, events, core-components, sling-models, htl, javascript, spa, debugging
---

## Adobe Client Data Layer (ACDL) — Complete Implementation Guide

The Adobe Client Data Layer (ACDL) is the canonical data layer for AEM as a Cloud Service. It is a JSON-based, event-driven, component-scoped state manager that lives at `window.adobeDataLayer`. Every analytics integration in AEM should read from and write to this layer.

---

### 1. ACDL Architecture

#### 1.1 What the Data Layer IS

The ACDL is NOT a plain JavaScript object. It is a **managed array** (`window.adobeDataLayer`) that:

- Accepts **pushes** of state objects and event objects
- Maintains a **computed state** — a deep-merged snapshot of all state pushes
- Records an **event history** — every push with an `event` property is recorded
- Provides **`getState()`** — returns the computed state (or a subtree of it)
- Supports **event listeners** — registered via `push(function(dl) { dl.addEventListener(...) })`
- Is **component-scoped** — each component contributes its own keyed subtree to the state

Under the hood the ACDL library (`@adobe/adobe-client-data-layer`) processes the array, merges state, dispatches events, and exposes the `getState()` and `addEventListener()` API on the array instance itself.

#### 1.2 Global Object

```javascript
// The data layer MUST be initialized as an array before the ACDL library loads.
// Core Components' page component does this automatically. For safety, always guard:
window.adobeDataLayer = window.adobeDataLayer || [];
```

Once the ACDL library loads (bundled in Core Components clientlib `core.wcm.components.commons.datalayer.v1`), the array is upgraded with:

- `adobeDataLayer.getState()` — returns full computed state
- `adobeDataLayer.getState(path)` — returns state subtree at dot-notation path
- `adobeDataLayer.addEventListener(eventName, callback)` — registers a listener
- `adobeDataLayer.removeEventListener(eventName, callback)` — removes a listener

#### 1.3 State Management Model

```
Push #1: { page: { "page-abc": { "dc:title": "Home" } } }
Push #2: { component: { "hero-xyz": { "@type": "myproject/components/hero" } } }
Push #3: { component: { "hero-xyz": { "dc:title": "Welcome" } } }   ← deep-merged into push #2

Computed state after all 3 pushes:
{
  "page": {
    "page-abc": { "dc:title": "Home" }
  },
  "component": {
    "hero-xyz": {
      "@type": "myproject/components/hero",
      "dc:title": "Welcome"
    }
  }
}
```

State is **deep-merged** — subsequent pushes with the same key path merge into existing state, they do not replace it (unless you explicitly set a value to `null` to delete it).

#### 1.4 Enabling the Data Layer

The data layer is controlled by a Sling Context-Aware Configuration.

**Configuration node (create if it does not exist):**

```
/conf/<mysite>/sling:configs/com.adobe.cq.wcm.core.components.internal.DataLayerConfig
    - jcr:primaryType = nt:unstructured
    - enabled = {Boolean}true
```

**Site root must reference the config:**

```
/content/<mysite>
    - sling:configRef = /conf/<mysite>
```

**Verification:** When the data layer is active, Core Components' page component renders:

```html
<body data-cmp-data-layer-enabled>
    ...
</body>
```

Check in browser console:

```javascript
// If this returns an object (not undefined), the data layer is active
window.adobeDataLayer.getState();
```

Projects generated with AEM Archetype v24+ have the data layer enabled by default.

#### 1.5 Core Components Integration

When enabled, every Core Component automatically:

1. Renders a `data-cmp-data-layer` attribute on its root element containing a JSON string with component metadata
2. Pushes its state into `window.adobeDataLayer` on page load
3. Fires `cmp:show`, `cmp:click`, `cmp:hide`, `cmp:loaded` events as appropriate

No custom code is needed for out-of-the-box tracking of Core Components.

---

### 2. Data Layer State Structure

#### 2.1 Page Data Schema

Every page component pushes its data under the `page` key, keyed by a generated page ID:

```json
{
  "page": {
    "page-5f7a8c3e2b": {
      "@type": "myproject/components/page",
      "repo:modifyDate": "2025-06-10T14:30:00Z",
      "dc:title": "Product Catalog",
      "dc:description": "Browse our full range of products",
      "xdm:tags": ["products", "catalog", "e-commerce"],
      "repo:path": "/content/mysite/en/products.html",
      "xdm:template": "/conf/mysite/settings/wcm/templates/product-listing-page",
      "xdm:language": "en-US"
    }
  }
}
```

| Property | Type | Description |
|----------|------|-------------|
| `@type` | String | Component resource type (sling:resourceType) |
| `repo:modifyDate` | ISO 8601 | Last modification date of the page resource |
| `dc:title` | String | Page title (jcr:title) |
| `dc:description` | String | Page description |
| `xdm:tags` | String[] | cq:tags assigned to the page |
| `repo:path` | String | Content path of the page |
| `xdm:template` | String | Template path |
| `xdm:language` | String | Language from jcr:language or path |

#### 2.2 Component Data Schema

Each component pushes its data under the `component` key, keyed by a generated component ID:

```json
{
  "component": {
    "teaser-9a2f4e7d1c": {
      "@type": "myproject/components/teaser",
      "dc:title": "Summer Sale",
      "dc:description": "Save up to 50% on selected items",
      "xdm:linkURL": "/content/mysite/en/promotions/summer-sale.html",
      "parentId": "page-5f7a8c3e2b"
    }
  }
}
```

| Property | Type | Description |
|----------|------|-------------|
| `@type` | String | Component resource type |
| `dc:title` | String | Component title (varies by component) |
| `dc:description` | String | Component description |
| `xdm:linkURL` | String | Primary link URL (teasers, buttons, etc.) |
| `parentId` | String | ID of the parent component or page |

**Image component data includes an `image` subobject:**

```json
{
  "component": {
    "image-3b8d5e1a9f": {
      "@type": "myproject/components/image",
      "dc:title": "Hero Banner",
      "image": {
        "repo:id": "dam-asset-id-123",
        "repo:modifyDate": "2025-03-20T09:15:00Z",
        "@type": "image/webp",
        "repo:path": "/content/dam/mysite/images/hero-banner.webp",
        "xdm:tags": ["hero", "homepage"]
      },
      "parentId": "page-5f7a8c3e2b"
    }
  }
}
```

#### 2.3 Full Page State Example

A complete page with multiple components:

```json
{
  "page": {
    "page-5f7a8c3e2b": {
      "@type": "myproject/components/page",
      "dc:title": "Product Catalog",
      "dc:description": "Browse our full range of products",
      "repo:path": "/content/mysite/en/products.html",
      "xdm:template": "/conf/mysite/settings/wcm/templates/product-listing-page",
      "xdm:language": "en-US",
      "repo:modifyDate": "2025-06-10T14:30:00Z",
      "xdm:tags": ["products", "catalog"]
    }
  },
  "component": {
    "title-a1b2c3d4e5": {
      "@type": "myproject/components/title",
      "dc:title": "Our Products",
      "parentId": "page-5f7a8c3e2b"
    },
    "image-3b8d5e1a9f": {
      "@type": "myproject/components/image",
      "dc:title": "Category Banner",
      "image": {
        "repo:id": "dam-456",
        "repo:modifyDate": "2025-05-01T08:00:00Z",
        "@type": "image/jpeg",
        "repo:path": "/content/dam/mysite/images/category-banner.jpg",
        "xdm:tags": []
      },
      "parentId": "page-5f7a8c3e2b"
    },
    "teaser-9a2f4e7d1c": {
      "@type": "myproject/components/teaser",
      "dc:title": "Summer Sale",
      "dc:description": "Save up to 50%",
      "xdm:linkURL": "/content/mysite/en/promotions/summer-sale.html",
      "parentId": "page-5f7a8c3e2b"
    },
    "tabs-f6g7h8i9j0": {
      "@type": "myproject/components/tabs",
      "dc:title": "Product Categories",
      "parentId": "page-5f7a8c3e2b",
      "shownItems": ["tabs-f6g7h8i9j0-item-electronics"]
    },
    "accordion-k1l2m3n4o5": {
      "@type": "myproject/components/accordion",
      "dc:title": "FAQ",
      "parentId": "page-5f7a8c3e2b",
      "shownItems": []
    }
  }
}
```

#### 2.4 Component ID Generation

Component IDs are generated by `ComponentUtils.generateId()` using the component resource type and resource path, producing a deterministic hash. The format is typically `<componentName>-<hash>`, e.g., `teaser-9a2f4e7d1c`.

---

### 3. Pushing Data from Code — Sling Model / Java

#### 3.1 Core Interface: ComponentData

All data layer integration in Sling Models revolves around `com.adobe.cq.wcm.core.components.models.datalayer.ComponentData`:

```java
public interface ComponentData {
    String getId();           // Unique component ID
    String getType();         // sling:resourceType
    String getTitle();        // dc:title
    String getDescription();  // dc:description
    String getLinkUrl();      // xdm:linkURL
    String getParentId();     // Parent component/page ID
    String getJson();         // Full JSON string for data-cmp-data-layer attribute
}
```

#### 3.2 DataLayerBuilder Utility

`com.adobe.cq.wcm.core.components.models.datalayer.DataLayerBuilder` provides a fluent API:

```java
import com.adobe.cq.wcm.core.components.models.datalayer.DataLayerBuilder;
import com.adobe.cq.wcm.core.components.util.ComponentUtils;

ComponentData data = DataLayerBuilder.forComponent()
    .withId(() -> ComponentUtils.generateId(resourceType, resource.getPath()))
    .withType(() -> resourceType)
    .withTitle(() -> title)
    .withDescription(() -> description)
    .withLinkUrl(() -> linkUrl)
    .withParentId(() -> ComponentUtils.getParentId(resource))
    .build();
```

Each `.with*()` method accepts a `Supplier<String>` (lambda). This defers evaluation until `build()` is called.

#### 3.3 Full Custom Component Example

**Interface:**

```java
package com.myproject.core.models;

import com.adobe.cq.wcm.core.components.models.datalayer.ComponentData;
import org.osgi.annotation.versioning.ConsumerType;

@ConsumerType
public interface ProductCard {
    String getProductName();
    String getProductSku();
    String getProductCategory();
    double getProductPrice();
    String getProductUrl();
    ComponentData getData();
}
```

**Implementation with data layer:**

```java
package com.myproject.core.models.impl;

import com.adobe.cq.export.json.ComponentExporter;
import com.adobe.cq.export.json.ExporterConstants;
import com.adobe.cq.wcm.core.components.models.datalayer.ComponentData;
import com.adobe.cq.wcm.core.components.models.datalayer.DataLayerBuilder;
import com.adobe.cq.wcm.core.components.models.datalayer.jackson.ComponentDataModelSerializer;
import com.adobe.cq.wcm.core.components.util.ComponentUtils;
import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import com.myproject.core.models.ProductCard;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.models.annotations.DefaultInjectionStrategy;
import org.apache.sling.models.annotations.Exporter;
import org.apache.sling.models.annotations.Model;
import org.apache.sling.models.annotations.injectorspecific.Self;
import org.apache.sling.models.annotations.injectorspecific.ValueMapValue;

@Model(
    adaptables = SlingHttpServletRequest.class,
    adapters = { ProductCard.class, ComponentExporter.class },
    resourceType = ProductCardImpl.RESOURCE_TYPE,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
@Exporter(
    name = ExporterConstants.SLING_MODEL_EXPORTER_NAME,
    extensions = ExporterConstants.SLING_MODEL_EXTENSION
)
public class ProductCardImpl implements ProductCard {

    protected static final String RESOURCE_TYPE = "myproject/components/product-card";

    @Self
    private Resource resource;

    @ValueMapValue
    private String productName;

    @ValueMapValue
    private String productSku;

    @ValueMapValue
    private String productCategory;

    @ValueMapValue
    private double productPrice;

    @ValueMapValue
    private String productUrl;

    @Override
    public String getProductName() {
        return productName;
    }

    @Override
    public String getProductSku() {
        return productSku;
    }

    @Override
    public String getProductCategory() {
        return productCategory;
    }

    @Override
    public double getProductPrice() {
        return productPrice;
    }

    @Override
    public String getProductUrl() {
        return productUrl;
    }

    @Override
    @JsonProperty("dataLayer")
    @JsonSerialize(using = ComponentDataModelSerializer.class)
    public ComponentData getData() {
        if (!ComponentUtils.isDataLayerEnabled(this.resource)) {
            return null;
        }
        return DataLayerBuilder.forComponent()
            .withId(() -> ComponentUtils.generateId(
                RESOURCE_TYPE, this.resource.getPath()))
            .withType(() -> RESOURCE_TYPE)
            .withTitle(this::getProductName)
            .withDescription(() -> "SKU: " + this.productSku
                + " | Category: " + this.productCategory)
            .withLinkUrl(this::getProductUrl)
            .withParentId(() -> ComponentUtils.getParentId(this.resource))
            .build();
    }

    @Override
    public String getExportedType() {
        return RESOURCE_TYPE;
    }
}
```

#### 3.4 Adding Custom Properties to Component Data

The standard `ComponentData` interface only supports a fixed set of properties. To add custom properties (like `productSku`, `productPrice`), extend `ComponentData`:

```java
package com.myproject.core.models.datalayer;

import com.adobe.cq.wcm.core.components.models.datalayer.ComponentData;
import com.fasterxml.jackson.annotation.JsonProperty;

public interface ProductComponentData extends ComponentData {

    @JsonProperty("xdm:sku")
    String getSku();

    @JsonProperty("xdm:price")
    Double getPrice();

    @JsonProperty("xdm:currency")
    String getCurrency();

    @JsonProperty("xdm:category")
    String getCategory();
}
```

```java
package com.myproject.core.models.datalayer.impl;

import com.adobe.cq.wcm.core.components.internal.models.v1.datalayer.ComponentDataImpl;
import com.adobe.cq.wcm.core.components.models.datalayer.ComponentData;
import com.myproject.core.models.datalayer.ProductComponentData;

public class ProductComponentDataImpl extends ComponentDataImpl implements ProductComponentData {

    private final String sku;
    private final Double price;
    private final String currency;
    private final String category;

    public ProductComponentDataImpl(ComponentData base, String sku,
                                     Double price, String currency, String category) {
        // Copy base properties
        super(base);
        this.sku = sku;
        this.price = price;
        this.currency = currency;
        this.category = category;
    }

    @Override
    public String getSku() {
        return sku;
    }

    @Override
    public Double getPrice() {
        return price;
    }

    @Override
    public String getCurrency() {
        return currency;
    }

    @Override
    public String getCategory() {
        return category;
    }
}
```

Then in the Sling Model, return the extended data:

```java
@Override
public ComponentData getData() {
    if (!ComponentUtils.isDataLayerEnabled(this.resource)) {
        return null;
    }
    ComponentData base = DataLayerBuilder.forComponent()
        .withId(() -> ComponentUtils.generateId(RESOURCE_TYPE, resource.getPath()))
        .withType(() -> RESOURCE_TYPE)
        .withTitle(this::getProductName)
        .withParentId(() -> ComponentUtils.getParentId(resource))
        .build();

    return new ProductComponentDataImpl(base, productSku, productPrice, "USD", productCategory);
}
```

The resulting JSON in `data-cmp-data-layer` will include:

```json
{
  "productcard-a1b2c3d4e5": {
    "@type": "myproject/components/product-card",
    "dc:title": "Running Shoes X500",
    "xdm:sku": "RS-X500",
    "xdm:price": 129.99,
    "xdm:currency": "USD",
    "xdm:category": "Footwear",
    "parentId": "page-9f8e7d6c5b"
  }
}
```

#### 3.5 Check Data Layer Enabled

Always guard data layer generation:

```java
// CORRECT: Guard with isDataLayerEnabled
@Override
public ComponentData getData() {
    if (ComponentUtils.isDataLayerEnabled(this.resource)) {
        return DataLayerBuilder.forComponent()
            .withId(/* ... */)
            .build();
    }
    return null;  // Data layer disabled — return null, HTL will skip the attribute
}

// WRONG: Always returning data, ignoring configuration
@Override
public ComponentData getData() {
    return DataLayerBuilder.forComponent()
        .withId(/* ... */)
        .build();
}
```

---

### 4. Pushing Data from Code — HTL

#### 4.1 Auto from Core Components

Core Components that extend `AbstractComponentImpl` or implement `ComponentExporter` with `getData()` automatically render the `data-cmp-data-layer` attribute. The component HTL template typically includes:

```html
<div data-cmp-data-layer="${component.data.json}">
    ...
</div>
```

#### 4.2 Custom Component HTL with Sling Model Data Layer

```html
<sly data-sly-use.card="com.myproject.core.models.ProductCard">
    <div class="cmp-product-card"
         data-cmp-data-layer="${card.data.json}"
         data-sly-test="${card.data}">

        <h3 class="cmp-product-card__name"
            data-cmp-clickable>${card.productName}</h3>

        <span class="cmp-product-card__price">
            $${card.productPrice}
        </span>

        <a class="cmp-product-card__link"
           href="${card.productUrl}"
           data-cmp-clickable>
            View Product
        </a>
    </div>
</sly>
```

Key points:
- `data-cmp-data-layer="${card.data.json}"` — outputs the JSON string from `ComponentData.getJson()`
- `data-sly-test="${card.data}"` — only renders the attribute when data layer is enabled (getData returns non-null)
- `data-cmp-clickable` — marks elements that should trigger `cmp:click` events when clicked

#### 4.3 Manual Data Layer JSON in HTL (Without Sling Model)

For simple cases where a Sling Model is not justified:

```html
<sly data-sly-use.compId="${'com.adobe.cq.wcm.core.components.util.ComponentUtils' @ resource=resource}">
    <div class="cmp-banner"
         data-cmp-data-layer='${ "{\"" + compId + "\":{\"@type\":\"myproject/components/banner\",\"dc:title\":\"" + properties.jcr__title + "\"}}" }'>
        <h2>${properties.jcr:title}</h2>
    </div>
</sly>
```

This approach is fragile and not recommended. Prefer using a Sling Model with `DataLayerBuilder`.

#### 4.4 Clickable Elements

Add `data-cmp-clickable` to any element that should fire a `cmp:click` event:

```html
<a href="${teaser.linkURL}"
   class="cmp-teaser__action"
   data-cmp-clickable>
    ${teaser.callToAction}
</a>

<button class="cmp-card__button"
        data-cmp-clickable>
    Add to Cart
</button>
```

The ACDL library automatically attaches click handlers to elements with `data-cmp-clickable` that are inside a container with `data-cmp-data-layer`. The `cmp:click` event includes `eventInfo.path` pointing to the parent component's data layer entry.

---

### 5. Pushing Data from Code — JavaScript (Client-Side)

#### 5.1 Pushing State Changes

```javascript
// Push component state — merges into computed state
window.adobeDataLayer = window.adobeDataLayer || [];
window.adobeDataLayer.push({
    component: {
        'search-results-a1b2c3': {
            '@type': 'myproject/components/search-results',
            'dc:title': 'Search Results',
            'xdm:totalResults': 42,
            'xdm:query': 'running shoes'
        }
    }
});

// Update existing component state — deep merged
window.adobeDataLayer.push({
    component: {
        'search-results-a1b2c3': {
            'xdm:totalResults': 38,          // Updated
            'xdm:appliedFilter': 'size:10'   // New property added
            // @type, dc:title, xdm:query remain from previous push
        }
    }
});

// Remove a state value by setting it to null
window.adobeDataLayer.push({
    component: {
        'search-results-a1b2c3': {
            'xdm:appliedFilter': null  // Deleted from state
        }
    }
});
```

#### 5.2 Pushing Events

Events require an `event` property. Optionally include `eventInfo` for context:

```javascript
// Push an event with associated state update
window.adobeDataLayer.push({
    event: 'cmp:click',
    eventInfo: {
        path: 'component.button-x7y8z9'
    },
    component: {
        'button-x7y8z9': {
            '@type': 'myproject/components/button',
            'dc:title': 'Add to Cart',
            'xdm:linkURL': '/api/cart/add'
        }
    }
});

// Push an event without state change (event-only)
window.adobeDataLayer.push({
    event: 'user:login',
    eventInfo: {
        path: 'user',
        method: 'email'
    }
});
```

#### 5.3 Reading Current State

```javascript
// Read full computed state
var fullState = window.adobeDataLayer.getState();

// Read page data only
var pageState = window.adobeDataLayer.getState('page');
// Returns: { "page-5f7a8c3e2b": { "@type": "...", "dc:title": "...", ... } }

// Read specific component by full path
var teaserData = window.adobeDataLayer.getState('component.teaser-9a2f4e7d1c');
// Returns: { "@type": "myproject/components/teaser", "dc:title": "Summer Sale", ... }

// Helper to get page title
function getPageTitle() {
    var pageState = window.adobeDataLayer.getState('page');
    if (pageState) {
        var pageId = Object.keys(pageState)[0];
        return pageState[pageId]['dc:title'];
    }
    return document.title;
}
```

#### 5.4 Event Listener Registration

Listeners MUST be registered via `push(function)` to ensure they work even if the data layer is not yet initialized:

```javascript
// CORRECT: Register via push(function) — works even if ACDL library hasn't loaded yet
window.adobeDataLayer = window.adobeDataLayer || [];
window.adobeDataLayer.push(function(dl) {
    dl.addEventListener('cmp:click', function(event) {
        console.log('Component clicked:', event.eventInfo.path);
    });
});

// WRONG: Direct call — fails if ACDL library hasn't loaded yet
window.adobeDataLayer.addEventListener('cmp:click', function(event) {
    // TypeError: addEventListener is not a function (if library not loaded)
});
```

The `push(function)` pattern queues the function. When the ACDL library loads, it processes queued functions and passes the upgraded data layer instance as the argument. If the library has already loaded, the function executes immediately.

#### 5.5 Event Types Reference

**Built-in Core Component events:**

| Event | Triggered When | Components |
|-------|---------------|------------|
| `cmp:loaded` | Data layer is fully populated after page load | Page component |
| `cmp:click` | User clicks an element with `data-cmp-clickable` | Any component with clickable elements |
| `cmp:show` | A panel/item becomes visible | Accordion, Tabs, Carousel |
| `cmp:hide` | A panel/item is hidden | Accordion, Tabs, Carousel |

**Meta event (catches all events):**

| Event | Triggered When |
|-------|---------------|
| `adobeDataLayer:event` | ANY event is pushed — acts as a wildcard listener |
| `adobeDataLayer:change` | Any state change (push without event, or with event that includes state) |

**Listener examples for each:**

```javascript
window.adobeDataLayer = window.adobeDataLayer || [];
window.adobeDataLayer.push(function(dl) {

    // Page loaded — fires once per page load
    dl.addEventListener('cmp:loaded', function(event) {
        var pageState = dl.getState('page');
        var pageId = Object.keys(pageState)[0];
        var page = pageState[pageId];
        console.log('[ACDL] Page loaded:', page['dc:title'], page['repo:path']);
    });

    // Click on any component with data-cmp-clickable
    dl.addEventListener('cmp:click', function(event) {
        var componentData = dl.getState(event.eventInfo.path);
        console.log('[ACDL] Click:', componentData['@type'], componentData['dc:title']);
    });

    // Component becomes visible (accordion panel opened, tab shown, carousel slide shown)
    dl.addEventListener('cmp:show', function(event) {
        var componentData = dl.getState(event.eventInfo.path);
        console.log('[ACDL] Show:', componentData['dc:title']);
    });

    // Component hidden (accordion panel closed, tab deselected)
    dl.addEventListener('cmp:hide', function(event) {
        var componentData = dl.getState(event.eventInfo.path);
        console.log('[ACDL] Hide:', componentData['dc:title']);
    });

    // Wildcard — listen to ALL events
    dl.addEventListener('adobeDataLayer:event', function(event) {
        console.log('[ACDL] Event:', event.event, event);
    });

    // State change listener — fires when computed state is modified
    dl.addEventListener('adobeDataLayer:change', function(event) {
        console.log('[ACDL] State changed:', event);
    });
});
```

#### 5.6 One-Time Listeners

To listen for an event only once (e.g., first page load), use the options parameter:

```javascript
window.adobeDataLayer.push(function(dl) {
    dl.addEventListener('cmp:loaded', function(event) {
        console.log('Page loaded — this fires only once');
    }, { once: true });
});
```

#### 5.7 Scoped Listeners

Listen for events only from a specific component by specifying a scope/path:

```javascript
window.adobeDataLayer.push(function(dl) {
    dl.addEventListener('cmp:click', function(event) {
        console.log('This specific button was clicked');
    }, { path: 'component.button-a1b2c3d4' });
});
```

#### 5.8 Past Event Replay

By default, listeners registered via `push(function)` will NOT replay past events. To also receive events that occurred before the listener was registered, set `scope` to `past`:

```javascript
window.adobeDataLayer.push(function(dl) {
    dl.addEventListener('cmp:loaded', function(event) {
        // This will fire even if cmp:loaded already happened before this code ran
        console.log('Page loaded (including past events)');
    }, { scope: 'past' });
});
```

Scope options:
- `'future'` (default) — only future events
- `'past'` — only past events
- `'all'` — both past and future events

---

### 6. Custom Events — Detailed Implementation

#### 6.1 Button Click with Custom Data

```javascript
document.querySelectorAll('[data-track-button]').forEach(function(button) {
    button.addEventListener('click', function() {
        var buttonId = this.getAttribute('data-track-button');
        var buttonText = this.textContent.trim();
        var buttonHref = this.getAttribute('href') || '';

        window.adobeDataLayer.push({
            event: 'cta:click',
            eventInfo: {
                path: 'component.' + buttonId,
                type: 'myproject/components/button'
            },
            component: {
                [buttonId]: {
                    '@type': 'myproject/components/button',
                    'dc:title': buttonText,
                    'xdm:linkURL': buttonHref,
                    'xdm:position': this.closest('section')
                        ? this.closest('section').getAttribute('data-section-name')
                        : 'unknown'
                }
            }
        });
    });
});
```

#### 6.2 Form Submission

```javascript
document.querySelectorAll('form[data-track-form]').forEach(function(form) {
    form.addEventListener('submit', function(e) {
        var formId = this.getAttribute('data-track-form');
        var formName = this.getAttribute('data-form-name') || this.getAttribute('name') || 'unknown';
        var formFields = Array.from(this.elements)
            .filter(function(el) { return el.name && el.type !== 'hidden' && el.type !== 'submit'; })
            .map(function(el) { return el.name; });

        window.adobeDataLayer.push({
            event: 'form:submit',
            eventInfo: {
                path: 'component.' + formId,
                type: 'myproject/components/form'
            },
            component: {
                [formId]: {
                    '@type': 'myproject/components/form',
                    'dc:title': formName,
                    'xdm:formName': formName,
                    'xdm:formFields': formFields,
                    'xdm:formFieldCount': formFields.length
                }
            }
        });
    });

    // Also track form field focus (for form abandonment analysis)
    form.querySelectorAll('input, select, textarea').forEach(function(field) {
        field.addEventListener('focus', function() {
            var formId = this.closest('form').getAttribute('data-track-form');
            window.adobeDataLayer.push({
                event: 'form:fieldFocus',
                eventInfo: {
                    path: 'component.' + formId,
                    fieldName: this.name
                }
            });
        });
    });
});
```

#### 6.3 Video Play / Pause / Complete

```javascript
function initVideoTracking(videoElement, videoId, videoTitle) {
    var milestones = [25, 50, 75];
    var milestonesReached = {};

    function pushVideoEvent(eventName, extraData) {
        var payload = {
            event: 'video:' + eventName,
            eventInfo: {
                path: 'component.' + videoId,
                type: 'myproject/components/video'
            },
            component: {
                [videoId]: Object.assign({
                    '@type': 'myproject/components/video',
                    'dc:title': videoTitle,
                    'xdm:duration': Math.round(videoElement.duration),
                    'xdm:currentTime': Math.round(videoElement.currentTime)
                }, extraData || {})
            }
        };
        window.adobeDataLayer.push(payload);
    }

    videoElement.addEventListener('play', function() {
        pushVideoEvent('play');
    });

    videoElement.addEventListener('pause', function() {
        pushVideoEvent('pause');
    });

    videoElement.addEventListener('ended', function() {
        pushVideoEvent('complete', { 'xdm:milestone': 100 });
    });

    videoElement.addEventListener('timeupdate', function() {
        if (!videoElement.duration) return;
        var percentPlayed = Math.floor(
            (videoElement.currentTime / videoElement.duration) * 100
        );
        milestones.forEach(function(milestone) {
            if (percentPlayed >= milestone && !milestonesReached[milestone]) {
                milestonesReached[milestone] = true;
                pushVideoEvent('milestone', { 'xdm:milestone': milestone });
            }
        });
    });
}

// Usage
document.querySelectorAll('video[data-track-video]').forEach(function(video) {
    initVideoTracking(
        video,
        video.getAttribute('data-track-video'),
        video.getAttribute('data-video-title') || 'Untitled Video'
    );
});
```

#### 6.4 Carousel Slide Changes

```javascript
// For Core Components Carousel — listen to cmp:show which fires automatically.
// For custom carousels, push manually:
function trackCarouselSlideChange(carouselId, slideIndex, slideTitle, totalSlides) {
    window.adobeDataLayer.push({
        event: 'carousel:slideChange',
        eventInfo: {
            path: 'component.' + carouselId,
            type: 'myproject/components/carousel'
        },
        component: {
            [carouselId]: {
                '@type': 'myproject/components/carousel',
                'dc:title': 'Product Carousel',
                'xdm:currentSlide': slideIndex,
                'xdm:slideTitle': slideTitle,
                'xdm:totalSlides': totalSlides
            }
        }
    });
}

// Assuming a custom carousel with a 'slideChanged' custom DOM event:
document.querySelector('.my-carousel').addEventListener('slideChanged', function(e) {
    trackCarouselSlideChange(
        'carousel-abc123',
        e.detail.index,
        e.detail.title,
        e.detail.total
    );
});
```

#### 6.5 Tab Switches

```javascript
// For Core Components Tabs — cmp:show and cmp:hide fire automatically.
// For custom tabs:
document.querySelectorAll('[data-tab-group]').forEach(function(tabGroup) {
    var groupId = tabGroup.getAttribute('data-tab-group');

    tabGroup.querySelectorAll('[data-tab-trigger]').forEach(function(trigger) {
        trigger.addEventListener('click', function() {
            var tabName = this.getAttribute('data-tab-trigger');
            var previousTab = tabGroup.querySelector('[data-tab-trigger].active');

            // Track hide of previous tab
            if (previousTab) {
                window.adobeDataLayer.push({
                    event: 'tab:hide',
                    eventInfo: {
                        path: 'component.' + groupId,
                        tabName: previousTab.getAttribute('data-tab-trigger')
                    }
                });
            }

            // Track show of new tab
            window.adobeDataLayer.push({
                event: 'tab:show',
                eventInfo: {
                    path: 'component.' + groupId,
                    tabName: tabName
                },
                component: {
                    [groupId]: {
                        '@type': 'myproject/components/tabs',
                        'xdm:activeTab': tabName
                    }
                }
            });
        });
    });
});
```

#### 6.6 Accordion Open / Close

```javascript
document.querySelectorAll('[data-accordion]').forEach(function(accordion) {
    var accordionId = accordion.getAttribute('data-accordion');

    accordion.querySelectorAll('[data-accordion-trigger]').forEach(function(trigger) {
        trigger.addEventListener('click', function() {
            var panelName = this.getAttribute('data-accordion-trigger');
            var isExpanding = !this.classList.contains('is-open');

            window.adobeDataLayer.push({
                event: isExpanding ? 'accordion:open' : 'accordion:close',
                eventInfo: {
                    path: 'component.' + accordionId,
                    panelName: panelName
                },
                component: {
                    [accordionId]: {
                        '@type': 'myproject/components/accordion',
                        'xdm:activePanel': isExpanding ? panelName : null
                    }
                }
            });
        });
    });
});
```

#### 6.7 Search Queries

```javascript
function trackSearch(query, resultCount, filters) {
    var searchId = 'search-' + Date.now().toString(36);

    window.adobeDataLayer.push({
        event: 'search:execute',
        eventInfo: {
            path: 'component.' + searchId,
            type: 'myproject/components/search'
        },
        component: {
            [searchId]: {
                '@type': 'myproject/components/search',
                'dc:title': 'Site Search',
                'xdm:query': query,
                'xdm:resultCount': resultCount,
                'xdm:hasResults': resultCount > 0,
                'xdm:filters': filters || {}
            }
        }
    });
}

// Usage: after search API returns results
searchApi.execute(query).then(function(results) {
    trackSearch(query, results.total, appliedFilters);
    renderResults(results);
});

// Track search result click
function trackSearchResultClick(query, resultPosition, resultTitle, resultUrl) {
    window.adobeDataLayer.push({
        event: 'search:resultClick',
        eventInfo: {
            path: 'component.search-result',
            type: 'myproject/components/search-result'
        },
        component: {
            'search-result': {
                '@type': 'myproject/components/search-result',
                'dc:title': resultTitle,
                'xdm:linkURL': resultUrl,
                'xdm:query': query,
                'xdm:position': resultPosition
            }
        }
    });
}
```

#### 6.8 Error States

```javascript
// Track application errors
function trackError(errorCode, errorMessage, errorSource) {
    window.adobeDataLayer.push({
        event: 'error:occur',
        eventInfo: {
            path: 'error',
            type: 'error'
        },
        error: {
            '@type': 'error',
            'xdm:errorCode': errorCode,
            'xdm:errorMessage': errorMessage,
            'xdm:errorSource': errorSource  // 'client', 'server', 'network'
        }
    });
}

// Track 404 page
if (document.querySelector('.cmp-error-page[data-error-code="404"]')) {
    window.adobeDataLayer.push({
        event: 'error:pageNotFound',
        eventInfo: {
            path: 'page',
            type: 'error'
        },
        error: {
            '@type': 'error',
            'xdm:errorCode': '404',
            'xdm:errorMessage': 'Page not found',
            'xdm:requestedUrl': window.location.pathname
        }
    });
}

// Track JavaScript runtime errors
window.addEventListener('error', function(e) {
    trackError('JS_ERROR', e.message, 'client');
});

// Track unhandled promise rejections
window.addEventListener('unhandledrejection', function(e) {
    trackError('PROMISE_REJECTION', e.reason ? e.reason.toString() : 'Unknown', 'client');
});
```

#### 6.9 Page Scroll Depth

```javascript
(function() {
    var scrollMilestones = [25, 50, 75, 90, 100];
    var milestonesReached = {};
    var ticking = false;

    function getScrollPercent() {
        var docHeight = document.documentElement.scrollHeight - window.innerHeight;
        if (docHeight <= 0) return 100;
        return Math.round((window.scrollY / docHeight) * 100);
    }

    function checkMilestones() {
        var percent = getScrollPercent();
        scrollMilestones.forEach(function(milestone) {
            if (percent >= milestone && !milestonesReached[milestone]) {
                milestonesReached[milestone] = true;
                window.adobeDataLayer.push({
                    event: 'page:scrollDepth',
                    eventInfo: {
                        path: 'page',
                        scrollPercent: milestone
                    }
                });
            }
        });
        ticking = false;
    }

    window.addEventListener('scroll', function() {
        if (!ticking) {
            ticking = true;
            requestAnimationFrame(checkMilestones);
        }
    }, { passive: true });
})();
```

---

### 7. Adobe Launch / Tags Integration

#### 7.1 Adobe Client Data Layer Extension

Install the **Adobe Client Data Layer** extension in your Adobe Experience Platform Tags (Launch) property. This extension:

- Automatically detects `window.adobeDataLayer`
- Provides "Data Layer Change" and "Data Layer Event" event types for rules
- Provides "Computed State" and "Event Data" data element types
- Handles past events if the Tags library loads after the ACDL

Configuration in the extension settings:
- **Data layer array name**: `adobeDataLayer` (default, rarely changed)

#### 7.2 Data Elements

Create data elements that read from the ACDL:

**Page Title data element:**
- Extension: Adobe Client Data Layer
- Type: Computed State
- Path: (optional — leave blank to get full state, or enter a path)
- In the custom code, access: `return this.getState('page')[Object.keys(this.getState('page'))[0]]['dc:title'];`

Alternatively, use simpler data element configurations:

| Data Element Name | Extension | Type | Path / Custom Code |
|---|---|---|---|
| ACDL - Page Title | Adobe Client Data Layer | Computed State | Custom code returning page title |
| ACDL - Page Path | Adobe Client Data Layer | Computed State | Custom code returning repo:path |
| ACDL - Page Template | Adobe Client Data Layer | Computed State | Custom code returning xdm:template |
| ACDL - Page Language | Adobe Client Data Layer | Computed State | Custom code returning xdm:language |
| ACDL - Component Type | Adobe Client Data Layer | Event Info | `eventInfo.type` or custom code |
| ACDL - Component Title | Adobe Client Data Layer | Computed State | Accessed from event context |
| ACDL - Click URL | Adobe Client Data Layer | Computed State | Read xdm:linkURL from component path |

**Custom code data element example (for Page Title):**

```javascript
// Data Element: ACDL - Page Title
// Extension: Adobe Client Data Layer
// Type: Computed State (with custom code)
var pageState = adobeDataLayer.getState('page');
if (pageState) {
    var pageId = Object.keys(pageState)[0];
    return pageState[pageId]['dc:title'] || '';
}
return '';
```

#### 7.3 Rules

**Rule: Track Page Views**

```
Event:
  Extension: Adobe Client Data Layer
  Event Type: Data Pushed
  Listen to: Specific Event
  Event/Key to Register For: cmp:loaded

Conditions: (none)

Actions:
  Action 1: Adobe Analytics - Set Variables
    - pageName = %ACDL - Page Title%
    - channel = %ACDL - Page Template%
    - prop1 = %ACDL - Page Language%
    - prop2 = %ACDL - Page Path%

  Action 2: Adobe Analytics - Send Beacon
    - Type: s.t() (page view)
```

**Rule: Track Component Clicks**

```
Event:
  Extension: Adobe Client Data Layer
  Event Type: Data Pushed
  Listen to: Specific Event
  Event/Key to Register For: cmp:click

Conditions: (none)

Actions:
  Action 1: Adobe Analytics - Set Variables
    - eVar10 = %ACDL - Component Type%
    - eVar11 = %ACDL - Component Title%
    - eVar12 = %ACDL - Click URL%
    - events = event10

  Action 2: Adobe Analytics - Send Beacon
    - Type: s.tl() (link tracking)
    - Link Name: %ACDL - Component Title%
```

**Rule: Track Tab/Accordion/Carousel Interactions**

```
Event:
  Extension: Adobe Client Data Layer
  Event Type: Data Pushed
  Listen to: Specific Event
  Event/Key to Register For: cmp:show

Conditions: (none)

Actions:
  Action 1: Adobe Analytics - Set Variables
    - eVar15 = %ACDL - Component Type%
    - eVar16 = %ACDL - Component Title%
    - events = event15

  Action 2: Adobe Analytics - Send Beacon
    - Type: s.tl() (link tracking)
    - Link Name: Component Shown - %ACDL - Component Title%
```

**Rule: Track Custom Search Event**

```
Event:
  Extension: Adobe Client Data Layer
  Event Type: Data Pushed
  Listen to: Specific Event
  Event/Key to Register For: search:execute

Actions:
  Action 1: Custom Code (JavaScript)
    var searchData = adobeDataLayer.getState(event.eventInfo.path);
    _satellite.setVar('searchQuery', searchData['xdm:query']);
    _satellite.setVar('searchResults', searchData['xdm:resultCount']);

  Action 2: Adobe Analytics - Set Variables
    - eVar20 = %searchQuery%
    - eVar21 = %searchResults%
    - events = event20

  Action 3: Adobe Analytics - Send Beacon
    - Type: s.tl()
    - Link Name: Internal Search
```

#### 7.4 Direct Call Rules vs ACDL Event Listeners

| Approach | When to Use |
|----------|-------------|
| ACDL Event Rules (via extension) | Standard component tracking, Core Component events, consistent patterns |
| Direct Call Rules (`_satellite.track()`) | One-off tracking needs, third-party widget callbacks, legacy migration |

Prefer ACDL event rules. They are decoupled from the tag manager and work consistently across environments.

---

### 8. Google Tag Manager Integration

#### 8.1 ACDL-to-GTM Bridge

GTM uses its own `window.dataLayer`. The bridge forwards ACDL events and state to GTM's format:

```javascript
// acdl-gtm-bridge.js — Load this BEFORE GTM snippet
(function() {
    'use strict';

    window.dataLayer = window.dataLayer || [];
    window.adobeDataLayer = window.adobeDataLayer || [];

    // Helper: get page data from ACDL state
    function getPageData(dl) {
        var pageState = dl.getState('page');
        if (!pageState) return {};
        var pageId = Object.keys(pageState)[0];
        return pageState[pageId] || {};
    }

    // Helper: get component data from event
    function getComponentData(dl, event) {
        if (event.eventInfo && event.eventInfo.path) {
            return dl.getState(event.eventInfo.path) || {};
        }
        return {};
    }

    window.adobeDataLayer.push(function(dl) {

        // Bridge: Page Load
        dl.addEventListener('cmp:loaded', function(event) {
            var page = getPageData(dl);
            window.dataLayer.push({
                event: 'aem.pageLoaded',
                aem: {
                    page: {
                        title: page['dc:title'] || '',
                        path: page['repo:path'] || '',
                        template: page['xdm:template'] || '',
                        language: page['xdm:language'] || '',
                        tags: page['xdm:tags'] || [],
                        description: page['dc:description'] || ''
                    }
                }
            });
        }, { scope: 'all' });

        // Bridge: Component Click
        dl.addEventListener('cmp:click', function(event) {
            var comp = getComponentData(dl, event);
            window.dataLayer.push({
                event: 'aem.componentClick',
                aem: {
                    component: {
                        type: comp['@type'] || '',
                        title: comp['dc:title'] || '',
                        linkUrl: comp['xdm:linkURL'] || '',
                        id: event.eventInfo ? event.eventInfo.path : ''
                    }
                }
            });
        });

        // Bridge: Component Show (tabs, accordion, carousel)
        dl.addEventListener('cmp:show', function(event) {
            var comp = getComponentData(dl, event);
            window.dataLayer.push({
                event: 'aem.componentShow',
                aem: {
                    component: {
                        type: comp['@type'] || '',
                        title: comp['dc:title'] || '',
                        id: event.eventInfo ? event.eventInfo.path : ''
                    }
                }
            });
        });

        // Bridge: Component Hide
        dl.addEventListener('cmp:hide', function(event) {
            var comp = getComponentData(dl, event);
            window.dataLayer.push({
                event: 'aem.componentHide',
                aem: {
                    component: {
                        type: comp['@type'] || '',
                        title: comp['dc:title'] || '',
                        id: event.eventInfo ? event.eventInfo.path : ''
                    }
                }
            });
        });

        // Bridge: Forward ALL custom events (catch-all)
        dl.addEventListener('adobeDataLayer:event', function(event) {
            // Skip built-in events already bridged above
            var builtIn = ['cmp:loaded', 'cmp:click', 'cmp:show', 'cmp:hide'];
            if (builtIn.indexOf(event.event) !== -1) return;

            window.dataLayer.push({
                event: 'aem.custom.' + event.event,
                aem: {
                    customEvent: event.event,
                    eventInfo: event.eventInfo || {},
                    state: event.eventInfo && event.eventInfo.path
                        ? dl.getState(event.eventInfo.path)
                        : null
                }
            });
        });
    });
})();
```

#### 8.2 GTM Triggers

Configure triggers in GTM to match the bridged events:

| Trigger Name | Trigger Type | Event Name | Purpose |
|---|---|---|---|
| AEM - Page Loaded | Custom Event | `aem.pageLoaded` | Page view tracking |
| AEM - Component Click | Custom Event | `aem.componentClick` | Click tracking |
| AEM - Component Show | Custom Event | `aem.componentShow` | Tab/accordion/carousel interaction |
| AEM - Form Submit | Custom Event | `aem.custom.form:submit` | Form submission tracking |
| AEM - Search | Custom Event | `aem.custom.search:execute` | Search tracking |
| AEM - Video Play | Custom Event | `aem.custom.video:play` | Video tracking |

#### 8.3 GTM Variables (Data Layer Variables)

| Variable Name | Type | Data Layer Variable Name |
|---|---|---|
| AEM Page Title | Data Layer Variable | `aem.page.title` |
| AEM Page Path | Data Layer Variable | `aem.page.path` |
| AEM Page Template | Data Layer Variable | `aem.page.template` |
| AEM Page Language | Data Layer Variable | `aem.page.language` |
| AEM Component Type | Data Layer Variable | `aem.component.type` |
| AEM Component Title | Data Layer Variable | `aem.component.title` |
| AEM Component Link URL | Data Layer Variable | `aem.component.linkUrl` |
| AEM Custom Event Name | Data Layer Variable | `aem.customEvent` |

#### 8.4 GTM Tag Example: Google Analytics 4

```
Tag: GA4 - Page View
  Type: Google Analytics: GA4 Event
  Configuration Tag: GA4 Configuration
  Event Name: page_view
  Event Parameters:
    - page_title: {{AEM Page Title}}
    - page_location: {{Page URL}}
    - page_path: {{AEM Page Path}}
    - content_group: {{AEM Page Template}}
    - language: {{AEM Page Language}}
  Trigger: AEM - Page Loaded

Tag: GA4 - Component Click
  Type: Google Analytics: GA4 Event
  Configuration Tag: GA4 Configuration
  Event Name: select_content
  Event Parameters:
    - content_type: {{AEM Component Type}}
    - item_id: {{AEM Component Title}}
    - link_url: {{AEM Component Link URL}}
  Trigger: AEM - Component Click
```

---

### 9. Debug Mode

#### 9.1 Console: Inspect Current State

```javascript
// Full state dump (formatted)
JSON.stringify(window.adobeDataLayer.getState(), null, 2);

// Page data only
JSON.stringify(window.adobeDataLayer.getState('page'), null, 2);

// Specific component
window.adobeDataLayer.getState('component.teaser-abc123');

// List all component IDs
Object.keys(window.adobeDataLayer.getState('component') || {});

// Get page title quickly
(function() {
    var p = window.adobeDataLayer.getState('page');
    return p ? p[Object.keys(p)[0]]['dc:title'] : 'N/A';
})();
```

#### 9.2 Console: Real-Time Event Monitor

Paste this in the console to log all ACDL events in real time:

```javascript
window.adobeDataLayer = window.adobeDataLayer || [];
window.adobeDataLayer.push(function(dl) {
    dl.addEventListener('adobeDataLayer:event', function(event) {
        var style = 'background:#1473e6;color:#fff;padding:2px 8px;border-radius:3px;';
        console.groupCollapsed('%c ACDL %c ' + event.event, style, '');
        if (event.eventInfo) {
            console.log('Path:', event.eventInfo.path || 'N/A');
            if (event.eventInfo.path) {
                console.log('Component state:', dl.getState(event.eventInfo.path));
            }
        }
        console.log('Full event object:', event);
        console.groupEnd();
    });
    console.log('%c ACDL event monitor active', 'color:#1473e6;font-weight:bold');
});
```

#### 9.3 Console: Event History

```javascript
// List all events that have been pushed
window.adobeDataLayer
    .filter(function(entry) { return typeof entry === 'object' && entry !== null && entry.event; })
    .map(function(entry) {
        return {
            event: entry.event,
            path: entry.eventInfo ? entry.eventInfo.path : 'N/A',
            timestamp: entry.timestamp || 'N/A'
        };
    });

// Count events by type
window.adobeDataLayer
    .filter(function(entry) { return typeof entry === 'object' && entry !== null && entry.event; })
    .reduce(function(acc, entry) {
        acc[entry.event] = (acc[entry.event] || 0) + 1;
        return acc;
    }, {});
```

#### 9.4 Console: Verify Data Layer Is Active

```javascript
// Quick health check
(function() {
    var checks = {
        'adobeDataLayer exists': !!window.adobeDataLayer,
        'adobeDataLayer is array': Array.isArray(window.adobeDataLayer),
        'getState available': typeof window.adobeDataLayer.getState === 'function',
        'addEventListener available': typeof window.adobeDataLayer.addEventListener === 'function',
        'body attribute present': document.body.hasAttribute('data-cmp-data-layer-enabled'),
        'has page state': !!(window.adobeDataLayer.getState && window.adobeDataLayer.getState('page')),
        'has component state': !!(window.adobeDataLayer.getState && window.adobeDataLayer.getState('component'))
    };
    console.table(checks);
    return checks;
})();
```

#### 9.5 Adobe Experience Platform Debugger

The **Adobe Experience Platform Debugger** browser extension (Chrome/Firefox):

- Shows all Tags (Launch) rule firings with matched conditions and actions
- Displays Analytics beacon data (s.t/s.tl calls with variable values)
- Shows Target requests, Audience Manager calls, and Web SDK events
- Provides a "Logs" tab showing the sequence of events and rule evaluations
- Can override the Tags environment (switch between Development/Staging/Production)

Use it to verify that ACDL events correctly trigger Launch rules and that the right variables are set.

#### 9.6 Chrome Extension: Adobe Data Layer Debugger

The community **Data Layer Debugger** Chrome extension provides a dedicated panel showing:

- Current computed state of `adobeDataLayer` as a tree
- Real-time event stream with filtering
- State diffs on each push

Useful during development when you need a persistent, always-visible view of the data layer.

---

### 10. Common Data Layer Patterns

#### 10.1 Page Load Tracking

The standard page view tracking pattern:

```javascript
window.adobeDataLayer = window.adobeDataLayer || [];
window.adobeDataLayer.push(function(dl) {
    dl.addEventListener('cmp:loaded', function(event) {
        var pageState = dl.getState('page');
        var pageId = Object.keys(pageState)[0];
        var page = pageState[pageId];

        analytics.trackPageView({
            pageName: page['dc:title'],
            pageUrl: page['repo:path'],
            template: page['xdm:template'],
            language: page['xdm:language'],
            tags: (page['xdm:tags'] || []).join(',')
        });
    }, { scope: 'all' });  // 'all' to catch the event even if it already fired
});
```

#### 10.2 Component Impression Tracking

Track when components scroll into view using IntersectionObserver:

```javascript
window.adobeDataLayer = window.adobeDataLayer || [];

(function() {
    var tracked = new Set();

    var observer = new IntersectionObserver(function(entries) {
        entries.forEach(function(entry) {
            if (!entry.isIntersecting) return;

            var el = entry.target;
            var dataLayerAttr = el.getAttribute('data-cmp-data-layer');
            if (!dataLayerAttr || tracked.has(el)) return;

            tracked.add(el);
            observer.unobserve(el);

            var componentData = JSON.parse(dataLayerAttr);
            var componentId = Object.keys(componentData)[0];

            window.adobeDataLayer.push({
                event: 'component:impression',
                eventInfo: {
                    path: 'component.' + componentId,
                    type: componentData[componentId]['@type']
                }
            });
        });
    }, {
        threshold: 0.5  // 50% visible
    });

    // Observe all components with data layer
    document.querySelectorAll('[data-cmp-data-layer]').forEach(function(el) {
        observer.observe(el);
    });
})();
```

#### 10.3 E-commerce Tracking

```javascript
// Product detail view
function trackProductView(product) {
    window.adobeDataLayer.push({
        event: 'product:view',
        eventInfo: {
            path: 'product',
            type: 'commerce'
        },
        product: {
            '@type': 'commerce/product',
            'xdm:SKU': product.sku,
            'xdm:name': product.name,
            'xdm:price': product.price,
            'xdm:currency': product.currency,
            'xdm:category': product.category,
            'xdm:brand': product.brand
        }
    });
}

// Add to cart
function trackAddToCart(product, quantity) {
    window.adobeDataLayer.push({
        event: 'cart:addItem',
        eventInfo: {
            path: 'cart',
            type: 'commerce'
        },
        cart: {
            '@type': 'commerce/cart',
            'xdm:SKU': product.sku,
            'xdm:name': product.name,
            'xdm:price': product.price,
            'xdm:quantity': quantity,
            'xdm:currency': product.currency
        }
    });
}

// Purchase complete
function trackPurchase(order) {
    window.adobeDataLayer.push({
        event: 'purchase:complete',
        eventInfo: {
            path: 'order',
            type: 'commerce'
        },
        order: {
            '@type': 'commerce/order',
            'xdm:orderId': order.id,
            'xdm:orderTotal': order.total,
            'xdm:currency': order.currency,
            'xdm:itemCount': order.items.length,
            'xdm:paymentMethod': order.paymentMethod,
            'xdm:items': order.items.map(function(item) {
                return {
                    'xdm:SKU': item.sku,
                    'xdm:name': item.name,
                    'xdm:price': item.price,
                    'xdm:quantity': item.quantity
                };
            })
        }
    });
}
```

#### 10.4 User Authentication State

```javascript
// Track login
function trackUserLogin(method) {
    window.adobeDataLayer.push({
        event: 'user:signIn',
        eventInfo: {
            path: 'user',
            type: 'identity'
        },
        user: {
            '@type': 'identity/user',
            'xdm:loginStatus': 'authenticated',
            'xdm:authMethod': method  // 'email', 'social', 'sso'
        }
    });
}

// Track logout
function trackUserLogout() {
    window.adobeDataLayer.push({
        event: 'user:signOut',
        eventInfo: {
            path: 'user',
            type: 'identity'
        },
        user: {
            '@type': 'identity/user',
            'xdm:loginStatus': 'anonymous'
        }
    });
}

// Push authentication state on page load (for returning authenticated users)
window.adobeDataLayer.push({
    user: {
        '@type': 'identity/user',
        'xdm:loginStatus': isAuthenticated ? 'authenticated' : 'anonymous',
        'xdm:memberTier': memberTier || 'none'
    }
});
```

#### 10.5 Search Results Tracking

```javascript
// Track internal search with results
function trackSearchResults(query, totalResults, page, resultsPerPage) {
    window.adobeDataLayer.push({
        event: 'search:results',
        eventInfo: {
            path: 'search',
            type: 'myproject/components/search'
        },
        search: {
            '@type': 'myproject/components/search',
            'xdm:query': query,
            'xdm:totalResults': totalResults,
            'xdm:page': page,
            'xdm:resultsPerPage': resultsPerPage,
            'xdm:hasResults': totalResults > 0
        }
    });
}

// Track zero results (separate event for easy rule matching)
function trackSearchNoResults(query) {
    window.adobeDataLayer.push({
        event: 'search:noResults',
        eventInfo: {
            path: 'search',
            type: 'myproject/components/search'
        },
        search: {
            '@type': 'myproject/components/search',
            'xdm:query': query,
            'xdm:totalResults': 0,
            'xdm:hasResults': false
        }
    });
}
```

#### 10.6 Error Page Tracking

```javascript
// Detect error page via component or HTTP status
window.adobeDataLayer = window.adobeDataLayer || [];
window.adobeDataLayer.push(function(dl) {
    dl.addEventListener('cmp:loaded', function() {
        var pageState = dl.getState('page');
        if (!pageState) return;
        var pageId = Object.keys(pageState)[0];
        var page = pageState[pageId];

        // Check if this is an error page template
        var errorTemplates = [
            '/conf/mysite/settings/wcm/templates/error-page',
            '/conf/mysite/settings/wcm/templates/404-page'
        ];
        if (errorTemplates.indexOf(page['xdm:template']) !== -1) {
            dl.push({
                event: 'error:pageView',
                eventInfo: {
                    path: 'page.' + pageId,
                    type: 'error'
                },
                error: {
                    '@type': 'error',
                    'xdm:errorCode': document.title.match(/404/) ? '404' : '500',
                    'xdm:requestedUrl': window.location.href,
                    'xdm:referrer': document.referrer
                }
            });
        }
    }, { scope: 'all' });
});
```

#### 10.7 SPA Navigation Tracking

For AEM SPA Editor sites using the Model Router:

```javascript
// In your SPA framework (React/Angular), listen for route changes
// and push new page state to the data layer

// React example with AEM SPA SDK
import { ModelManager } from '@adobe/aem-spa-page-model-manager';

ModelManager.addListener('modelUpdated', function(event) {
    if (event.path && event.model) {
        var pageData = event.model[':items'] && event.model[':items'].page
            ? event.model[':items'].page
            : event.model;

        // Generate a page ID from the path
        var pageId = 'page-' + event.path.replace(/[^a-zA-Z0-9]/g, '').substring(0, 10);

        window.adobeDataLayer.push({
            event: 'cmp:loaded',
            eventInfo: {
                path: 'page.' + pageId
            },
            page: {
                [pageId]: {
                    '@type': pageData[':type'] || 'myproject/components/page',
                    'dc:title': pageData['jcr:title'] || document.title,
                    'repo:path': event.path,
                    'xdm:language': document.documentElement.lang || 'en'
                }
            }
        });
    }
});

// Alternatively, hook into React Router or Angular Router
// React Router example:
import { useLocation } from 'react-router-dom';
import { useEffect } from 'react';

function usePageTracking() {
    var location = useLocation();

    useEffect(function() {
        // Small delay to let SPA content render and data layer update
        var timer = setTimeout(function() {
            var pageState = window.adobeDataLayer.getState('page');
            if (pageState) {
                window.adobeDataLayer.push({
                    event: 'spa:pageView',
                    eventInfo: {
                        path: 'page.' + Object.keys(pageState)[0],
                        url: window.location.href
                    }
                });
            }
        }, 100);
        return function() { clearTimeout(timer); };
    }, [location.pathname]);
}
```

---

### 11. Anti-Patterns

#### 11.1 Pushing Before Initialization

```javascript
// WRONG: calling getState() or addEventListener() directly on the array before
// the ACDL library has loaded — these methods don't exist on a plain array
window.adobeDataLayer.getState();  // TypeError: not a function

// CORRECT: always use push(function) pattern which queues until library loads
window.adobeDataLayer = window.adobeDataLayer || [];
window.adobeDataLayer.push(function(dl) {
    var state = dl.getState();  // Safe — library is guaranteed to be loaded
});
```

#### 11.2 Not Using the push(function) Pattern

```javascript
// WRONG: Registering listener directly — race condition if library not loaded
window.adobeDataLayer.addEventListener('cmp:click', handler);

// CORRECT: Always wrap in push(function)
window.adobeDataLayer.push(function(dl) {
    dl.addEventListener('cmp:click', handler);
});
```

#### 11.3 Directly Modifying the Array

```javascript
// WRONG: Directly manipulating the array — bypasses the ACDL processing engine
window.adobeDataLayer[0] = { page: { ... } };
window.adobeDataLayer.splice(0, 1);
window.adobeDataLayer.length = 0;  // "clearing" the data layer

// CORRECT: Only interact via push()
window.adobeDataLayer.push({ page: { ... } });
```

#### 11.4 Circular References

```javascript
// WRONG: Circular reference causes JSON.stringify to fail inside ACDL
var obj = { name: 'test' };
obj.self = obj;  // Circular!
window.adobeDataLayer.push({ component: { 'bad-comp': obj } });
// Results in: TypeError: Converting circular structure to JSON

// CORRECT: Always push plain, serializable objects
window.adobeDataLayer.push({
    component: {
        'good-comp': {
            '@type': 'myproject/components/test',
            'dc:title': 'test'
        }
    }
});
```

#### 11.5 Excessive Push Frequency

```javascript
// WRONG: Pushing on every scroll pixel — floods the data layer and kills performance
window.addEventListener('scroll', function() {
    window.adobeDataLayer.push({
        event: 'page:scroll',
        eventInfo: { scrollY: window.scrollY }
    });
});

// CORRECT: Throttle and use milestones
var scrollThrottle = null;
var lastMilestone = 0;
window.addEventListener('scroll', function() {
    if (scrollThrottle) return;
    scrollThrottle = setTimeout(function() {
        scrollThrottle = null;
        var percent = Math.round(
            (window.scrollY / (document.documentElement.scrollHeight - window.innerHeight)) * 100
        );
        var milestone = Math.floor(percent / 25) * 25;
        if (milestone > lastMilestone) {
            lastMilestone = milestone;
            window.adobeDataLayer.push({
                event: 'page:scrollDepth',
                eventInfo: { scrollPercent: milestone }
            });
        }
    }, 200);
}, { passive: true });
```

#### 11.6 Relying on Data Layer in SSR Without Client-Side Hydration

```javascript
// WRONG: Assuming server-rendered data-cmp-data-layer attributes are
// automatically in the ACDL state. They are NOT — the ACDL library must
// parse them client-side. If ACDL JS fails to load, getState() returns nothing.

// On SSR/Edge-cached pages, always ensure:
// 1. The ACDL library JS is loaded (core.wcm.components.commons.datalayer.v1 clientlib)
// 2. You do NOT call getState() before DOMContentLoaded
// 3. Your tracking code uses push(function) which waits for library readiness

// CORRECT: Guard all access
window.adobeDataLayer = window.adobeDataLayer || [];
window.adobeDataLayer.push(function(dl) {
    // Safe — library is loaded, DOM attributes have been parsed into state
    var state = dl.getState();
});
```

#### 11.7 Non-Standard Event Structure

```javascript
// WRONG: Missing event, eventInfo.path, and @type — cannot be matched by ACDL listeners
window.adobeDataLayer.push({
    action: 'buttonClick',
    buttonName: 'Submit'
});

// CORRECT: Follow the ACDL event convention
window.adobeDataLayer.push({
    event: 'button:click',
    eventInfo: {
        path: 'component.submit-button-123',
        type: 'myproject/components/button'
    },
    component: {
        'submit-button-123': {
            '@type': 'myproject/components/button',
            'dc:title': 'Submit'
        }
    }
});
```

#### 11.8 Pushing DOM Elements or Functions as State

```javascript
// WRONG: DOM elements and functions are not serializable
window.adobeDataLayer.push({
    component: {
        'widget-123': {
            element: document.getElementById('widget'),  // DOM node!
            handler: function() { return 'bad'; }         // Function!
        }
    }
});

// CORRECT: Only push serializable primitives, arrays, and plain objects
window.adobeDataLayer.push({
    component: {
        'widget-123': {
            '@type': 'myproject/components/widget',
            'dc:title': 'My Widget',
            'xdm:elementId': 'widget'  // Reference by ID string, not DOM node
        }
    }
});
```

---

### Pitfalls Summary

- **Never bypass ACDL** with inline `onclick` handlers or manual tracking for components that support the data layer
- **Never poll** the data layer — always use `addEventListener` via `push(function(dl) {...})`
- **Never call `getState()` or `addEventListener()` directly** on the array before the ACDL library has loaded — use `push(function)` to queue
- **Never forget** `ComponentUtils.isDataLayerEnabled()` in custom Sling Models
- **Never push non-serializable values** (DOM elements, functions, circular references) into the data layer
- **Always push custom events** with `event`, `eventInfo.path`, and `@type` for consistent rule matching
- **Always throttle** high-frequency events (scroll, mousemove, resize) before pushing to the data layer
- **Always use `scope: 'all'`** on `cmp:loaded` listeners to catch the event even if it already fired
- **Always verify** the data layer is active by checking `data-cmp-data-layer-enabled` on `<body>`
- **Use `getState(path)`** to read component data — do not parse the raw `data-cmp-data-layer` HTML attribute client-side
